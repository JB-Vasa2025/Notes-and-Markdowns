# Rockerbox Classification View

## Purpose
Maps every Rockerbox tier_1/tier_2/tier_3 combination to the dashboard's **Channel** and **Partner** dimensions. Also normalizes tier_4 (campaign name) for downstream joining to the Mapping Table.

This view handles:
- All paid channels (Paid Social, Search, Display, Demand Gen, PMAX, Paid Video)
- All organic channels (Email, SMS, Organic Search/Social, Direct, Referrer, etc.)
- Unmapped Events reclassification (platform integrations → proper paid channels)
- Old-format campaign name normalization alongside new VASA-format
- Data quality fixes (facebook→meta merge, case standardization, tier_1 consolidation)

---

## Dependencies
- **Source table:** `marketing.rockerbox_raw` — your Rockerbox MTA export
- **Expected columns:** `tier_1`, `tier_2`, `tier_3`, `tier_4`, `tier_5`, `total_joins`, `min_date`, `max_date`

---

## SQL

```sql
CREATE OR REPLACE VIEW marketing.rockerbox_classified AS

-- =====================================================
-- CTE 1: CLEAN & STANDARDIZE RAW TIERS
-- Fixes known data quality issues before mapping
-- =====================================================
WITH normalized AS (
  SELECT
    *,

    -- Consolidate tier_1 variants
    CASE
      WHEN tier_1 = 'Paid Search' THEN 'Search'                          -- Merge 1-row outlier
      WHEN tier_1 = 'Unmapped' AND tier_2 = 'the_trade_desk' THEN 'Display'  -- Reclassify TTD
      ELSE tier_1
    END AS tier_1_clean,

    -- Standardize tier_2 (lowercase + facebook→meta)
    CASE
      WHEN tier_2 = 'facebook' THEN 'meta'
      WHEN tier_1 = 'Unmapped' AND tier_2 = 'the_trade_desk' THEN 'ttd'
      ELSE LOWER(tier_2)
    END AS tier_2_clean,

    -- Standardize tier_3 casing
    INITCAP(LOWER(tier_3)) AS tier_3_clean

  FROM marketing.rockerbox_raw
),

-- =====================================================
-- CTE 2: MAP CHANNEL + PARTNER + NORMALIZE CAMPAIGN NAME
-- =====================================================
channel_mapped AS (
  SELECT
    *,

    -- -----------------------------------------------
    -- MAPPED CHANNEL
    -- -----------------------------------------------
    CASE
      -- Paid channels
      WHEN tier_1_clean = 'Paid Social'      THEN 'Paid Social'
      WHEN tier_1_clean = 'Display'          THEN 'Display'
      WHEN tier_1_clean = 'Paid Display'     THEN 'Demand Generation'
      WHEN tier_1_clean = 'Paid Video'       THEN 'Demand Generation'
      WHEN tier_1_clean = 'Performance Max'  THEN 'PMAX'
      WHEN tier_1_clean = 'Search'           THEN 'Paid Search'

      -- Organic channels
      WHEN tier_1_clean = 'Email'            THEN 'CRM - Email'
      WHEN tier_1_clean = 'SMS'              THEN 'CRM - SMS'
      WHEN tier_1_clean = 'Organic Search'   THEN 'Organic Search'
      WHEN tier_1_clean = 'Organic Social'   THEN 'Organic Social'
      WHEN tier_1_clean = 'Direct'           THEN 'Direct'
      WHEN tier_1_clean = 'Referrer'         THEN 'Referrer'
      WHEN tier_1_clean = 'Affiliate'        THEN 'Affiliate'
      WHEN tier_1_clean = 'Organic Shopping' THEN 'Organic Shopping'
      WHEN tier_1_clean = 'Customer Referral' THEN 'Customer Referral'

      -- Unmapped Events: reclassify paid integrations
      WHEN tier_1_clean = 'Unmapped Events' THEN
        CASE
          WHEN tier_2 LIKE '%Facebook%' OR tier_2 LIKE '%Instagram%' THEN 'Paid Social'
          WHEN tier_2 LIKE '%Snap%'    THEN 'Paid Social'
          WHEN tier_2 LIKE '%TikTok%'  THEN 'Paid Social'
          WHEN tier_2 LIKE '%Trade Desk%' THEN 'Display'
          WHEN tier_2 LIKE '%Google%'  THEN 'Paid Search'
          ELSE 'Other'
        END

      ELSE 'Other'
    END AS mapped_channel,

    -- -----------------------------------------------
    -- MAPPED PARTNER
    -- IMPORTANT: Search uses tier_2 + tier_3
    -- -----------------------------------------------
    CASE
      -- Paid Social: tier_2 = partner
      WHEN tier_1_clean = 'Paid Social' THEN
        CASE
          WHEN tier_2_clean IN ('meta', 'facebook') THEN 'Meta'
          WHEN tier_2_clean = 'tiktok'   THEN 'TikTok'
          WHEN tier_2_clean = 'snapchat' THEN 'Snapchat'
          ELSE INITCAP(tier_2_clean)
        END

      -- Display: always TTD
      WHEN tier_1_clean = 'Display' THEN 'TheTradeDesk'

      -- Demand Gen channels
      WHEN tier_1_clean = 'Paid Display' THEN 'Google'
      WHEN tier_1_clean = 'Paid Video'   THEN 'YouTube'

      -- PMAX: always Google PMAX
      WHEN tier_1_clean = 'Performance Max' THEN 'Google PMAX'

      -- Search: CRITICAL — Partner = tier_2 + tier_3
      WHEN tier_1_clean = 'Search' THEN
        CASE
          WHEN LOWER(tier_3) = 'brand'    THEN 'Google Brand'
          WHEN LOWER(tier_3) = 'nonbrand' THEN 'Google Non Brand'
          WHEN LOWER(tier_3) = 'dsa'      THEN 'Google DSA'
          WHEN LOWER(tier_3) = 'promo'    THEN 'Google Non Brand'
          ELSE 'Google ' || INITCAP(tier_3)
        END

      -- Email partners
      WHEN tier_1_clean = 'Email' THEN
        CASE
          WHEN tier_2_clean = 'hs_automation'  THEN 'HubSpot Automation'
          WHEN tier_2_clean IN ('hs_email', 'hubspot') THEN 'HubSpot Email'  -- hubspot = legacy label, same grouping
          WHEN tier_2_clean = 'listrak'         THEN 'Listrak'               -- legacy ESP, keep classified
          WHEN tier_2_clean = 'mailchimp'       THEN 'Mailchimp'             -- legacy ESP, keep classified
          ELSE 'HubSpot'
        END

      -- SMS partners
      WHEN tier_1_clean = 'SMS' THEN
        CASE
          WHEN tier_2_clean = 'pac' THEN 'PAC'
          ELSE 'HubSpot SMS'
        END

      -- Organic channels: tier_2 = partner
      WHEN tier_1_clean IN ('Organic Search', 'Organic Social') THEN INITCAP(tier_2_clean)

      -- Unmapped Events: parse integration name
      WHEN tier_1_clean = 'Unmapped Events' THEN
        CASE
          WHEN tier_2 LIKE '%Facebook%' OR tier_2 LIKE '%Instagram%' THEN 'Meta'
          WHEN tier_2 LIKE '%Snap%'       THEN 'Snapchat'
          WHEN tier_2 LIKE '%TikTok%'     THEN 'TikTok'
          WHEN tier_2 LIKE '%Trade Desk%' THEN 'TheTradeDesk'
          WHEN tier_2 LIKE '%Google%'     THEN 'Google'
          ELSE tier_2
        END

      ELSE COALESCE(INITCAP(tier_2_clean), 'Unknown')
    END AS mapped_partner,

    -- -----------------------------------------------
    -- CAMPAIGN NAME NORMALIZATION
    -- Handles both new VASA format and old formats
    -- -----------------------------------------------
    CASE
      -- New VASA format (e.g., vasa___colorado_prospectingoffers___sales_website____)
      WHEN tier_4 LIKE 'vasa\\_%' THEN
        LOWER(REGEXP_REPLACE(
          REGEXP_REPLACE(
            REGEXP_REPLACE(tier_4, '_{2,}', ' | '),
          '_', '-'),
        '\\|$|^\\|', ''))

      -- Old Search format (e.g., gs___brand_jun23___colorado____venuegroup_00285006_active_)
      WHEN tier_4 LIKE 'gs\\_\\_\\_%' THEN
        LOWER(REGEXP_REPLACE(
          REGEXP_REPLACE(
            REGEXP_REPLACE(tier_4, '____', ' | {'),
          '___', ' | '),
        '_active_$', ',Active}'))

      -- Old PMAX format (e.g., gp___evergreen___colorado____venuegroup_00285001_active_)
      WHEN tier_4 LIKE 'gp\\_\\_\\_%' THEN
        LOWER(REGEXP_REPLACE(
          REGEXP_REPLACE(
            REGEXP_REPLACE(tier_4, '____', ' | {'),
          '___', ' | '),
        '_active_$', ',Active}'))

      -- Old Demand Gen format (e.g., demand_gen___all_locations___retargeting___gmail_discover)
      -- Note: final REPLACE adds slash back for "gmail/discover" to match Mapping Table
      WHEN tier_4 LIKE 'demand\\_gen%' THEN
        LOWER(REPLACE(REPLACE(REPLACE(tier_4, '___', ' - '), '_', ' '), 'gmail discover', 'gmail/discover'))

      -- Old TikTok format (e.g., 271919___vasa_fitness___seasoned_clubs___conversions___july25)
      WHEN REGEXP_LIKE(tier_4, '^[0-9]{5,6}___') THEN
        LOWER(REPLACE(tier_4, '___', ' | '))

      -- Old Paid Social focus___flight format (e.g., prospecting_free_trial___july271918)
      WHEN REGEXP_LIKE(tier_4, '^(prospecting|retargeting|nco)') THEN
        LOWER(tier_4)

      -- A/B test prefix (e.g., test___vasa___seasonedclubsexcl_utahversiona_abtest___)
      -- Strip test___ prefix, then apply full VASA normalization to the remainder
      WHEN tier_4 LIKE 'test\\_\\_\\_%' THEN
        LOWER(REGEXP_REPLACE(
          REGEXP_REPLACE(
            REGEXP_REPLACE(
              REGEXP_REPLACE(tier_4, '^test___', ''),
              '_{2,}', ' | '),
            '_', '-'),
          '\\|$|^\\|', ''))

      -- Bare numeric IDs — cannot parse
      WHEN REGEXP_LIKE(tier_4, '^[0-9]+$') THEN NULL

      -- Everything else: lowercase as-is
      ELSE LOWER(tier_4)
    END AS tier_4_normalized,

    -- -----------------------------------------------
    -- CAMPAIGN FOCUS (from tier_3 for most channels)
    -- For Search, tier_3 is used for Partner, not focus
    -- -----------------------------------------------
    CASE
      WHEN tier_1_clean = 'Search' THEN NULL
      WHEN tier_3_clean IN ('Prospecting', 'Retargeting') THEN tier_3_clean
      WHEN tier_3_clean = 'Mixed' THEN 'Mixed'
      WHEN tier_3_clean = 'Ror Custom Audience' THEN 'Custom Audience'
      WHEN tier_3_clean = 'Employee Branding' THEN 'Employee Branding'
      ELSE tier_3_clean
    END AS mapped_campaign_focus

  FROM normalized
)

-- =====================================================
-- FINAL SELECT: Add campaign descriptor for Layer 2 joins
-- =====================================================
SELECT
  cm.*,

  CASE
    WHEN tier_4_normalized IS NOT NULL THEN
      CONCAT(
        SPLIT_PART(REPLACE(tier_4_normalized, ' | ', '___'), '___', 1),
        '___',
        SPLIT_PART(REPLACE(tier_4_normalized, ' | ', '___'), '___', 2)
      )
    ELSE NULL
  END AS campaign_descriptor

FROM channel_mapped cm;
```

---

## Output Columns

| Column | Description |
|--------|-------------|
| `tier_1` through `tier_5` | Original Rockerbox values (unchanged) |
| `tier_1_clean` | Standardized tier_1 (merges Paid Search→Search, reclassifies Unmapped TTD) |
| `tier_2_clean` | Lowercased tier_2 with facebook→meta merge |
| `tier_3_clean` | Title-cased tier_3 |
| `mapped_channel` | Dashboard Channel dimension (e.g., Paid Social, Paid Search, Display) |
| `mapped_partner` | Dashboard Partner dimension (e.g., Meta, Google Brand, TheTradeDesk) |
| `tier_4_normalized` | Normalized campaign name for joining to Mapping Table. NULL if unparseable. |
| `mapped_campaign_focus` | Campaign objective extracted from tier_3 (Prospecting, Retargeting, etc.) |
| `campaign_descriptor` | First two segments of normalized name — used for fuzzy campaign matching |

---

## Validation After Deployment

Run this to confirm all tier_1 values are classified:

```sql
SELECT mapped_channel, mapped_partner, COUNT(*) AS rows, SUM(total_joins) AS joins
FROM marketing.rockerbox_classified
GROUP BY 1, 2
ORDER BY joins DESC;
```

Every row should have a non-null `mapped_channel` and `mapped_partner`. If any show as "Other" / "Unknown", a new tier_1 or tier_2 value has appeared that needs a mapping rule added.
