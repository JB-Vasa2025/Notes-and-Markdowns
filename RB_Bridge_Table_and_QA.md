# Rockerbox → Mapping Table Bridge & QA

## Purpose
Joins the classified Rockerbox data (from `marketing.rockerbox_classified`) back to the paid media Mapping Table to pull in campaign-level dimensions: Campaign Grouping, Campaign Type, Region, State, and campaign_id. Unmatched rows are flagged as **"Unavailable Mapping"** — never dropped.

---

## Dependencies
- **Source view:** `marketing.rockerbox_classified` (created by [RB_Classification_View.md](./RB_Classification_View.md))
- **Mapping Table:** `marketing.mapping_table_campaigns` — the Campaign sheet from the Mapping Table Excel, loaded into Databricks
  - Expected columns: `Campaign`, `campaign_id`, `Source`, `Campaign_Grouping`, `Region`, `State`, `Campaign_Type`, `Campaign_Focus`, `Channel`, `Partner`, `Club_Grouping`

---

## Bridge Table SQL

```sql
CREATE OR REPLACE VIEW marketing.rb_to_platform_bridge AS

-- =====================================================
-- STEP 1: NORMALIZE MAPPING TABLE CAMPAIGN NAMES
-- Apply the same normalization as the RB side so both
-- sides are comparable for joining
-- =====================================================
WITH mt_normalized AS (
  SELECT
    *,

    -- Full normalized campaign name
    LOWER(
      REGEXP_REPLACE(
        REGEXP_REPLACE(
          REGEXP_REPLACE(
            REGEXP_REPLACE(Campaign, ' \\| ', '___'),
          '-', '_'),
        ' ', '_'),
      '[{}]', '')
    ) AS campaign_normalized,

    -- Campaign descriptor (first 2 segments only — ignores club IDs)
    CONCAT(
      SPLIT_PART(
        LOWER(REGEXP_REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(
          Campaign, ' \\| ', '___'), '-', '_'), ' ', '_'), '[{}]', '')),
        '___', 1),
      '___',
      SPLIT_PART(
        LOWER(REGEXP_REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(
          Campaign, ' \\| ', '___'), '-', '_'), ' ', '_'), '[{}]', '')),
        '___', 2)
    ) AS mt_descriptor

  FROM marketing.mapping_table_campaigns
),

-- =====================================================
-- STEP 2: JOIN ROCKERBOX TO MAPPING TABLE
-- Uses layered matching:
--   Layer 1: Channel + Partner (required)
--   Layer 2: Campaign descriptor OR exact campaign name
--   Fallback: Unavailable Mapping
-- =====================================================
joined AS (
  SELECT
    rb.*,
    mt.campaign_id,
    mt.Campaign          AS mt_campaign_name,
    mt.Campaign_Grouping AS mt_campaign_grouping,
    mt.Campaign_Type     AS mt_campaign_type,
    mt.Campaign_Focus    AS mt_campaign_focus,
    mt.Region            AS mt_region,
    mt.State             AS mt_state,
    mt.Club_Grouping     AS mt_club_grouping,
    mt.Source             AS mt_source,

    -- Track which join layer matched
    CASE
      WHEN mt.campaign_id IS NOT NULL
        AND rb.tier_4_normalized = mt.campaign_normalized
        THEN 'Exact Campaign Match'
      WHEN mt.campaign_id IS NOT NULL
        AND rb.campaign_descriptor = mt.mt_descriptor
        THEN 'Descriptor Match'
      WHEN mt.campaign_id IS NOT NULL
        THEN 'Channel+Partner Only'
      ELSE 'No Match'
    END AS join_method,

    -- Row number to deduplicate (prefer exact match, then descriptor)
    ROW_NUMBER() OVER (
      PARTITION BY rb.tier_1, rb.tier_2, rb.tier_3, rb.tier_4, rb.tier_5
      ORDER BY
        CASE
          WHEN rb.tier_4_normalized = mt.campaign_normalized THEN 1
          WHEN rb.campaign_descriptor = mt.mt_descriptor THEN 2
          ELSE 3
        END
    ) AS match_rank

  FROM marketing.rockerbox_classified rb

  LEFT JOIN mt_normalized mt
    ON rb.mapped_channel = mt.Channel
    AND rb.mapped_partner = mt.Partner
    AND (
      rb.tier_4_normalized = mt.campaign_normalized
      OR rb.campaign_descriptor = mt.mt_descriptor
    )
)

-- =====================================================
-- STEP 3: DEDUPLICATE + APPLY FALLBACK
-- Keep best match per RB row; flag unmatched as Unavailable
-- =====================================================
SELECT
  -- RB dimensions
  tier_1, tier_2, tier_3, tier_4, tier_5,
  total_joins, min_date, max_date,

  -- Mapped dimensions (from classification view)
  mapped_channel,
  mapped_partner,
  mapped_campaign_focus,
  tier_4_normalized,
  campaign_descriptor,

  -- Mapping Table dimensions (with Unavailable Mapping fallback)
  COALESCE(mt_campaign_grouping, 'Unavailable Mapping') AS campaign_grouping,
  COALESCE(mt_campaign_type, 'Unavailable Mapping')     AS campaign_type,
  COALESCE(mt_region, 'Unavailable Mapping')             AS region,
  COALESCE(mt_state, 'Unavailable Mapping')              AS state,
  COALESCE(mt_club_grouping, 'Unavailable Mapping')      AS club_grouping,

  -- Join metadata
  campaign_id          AS mt_campaign_id,
  mt_campaign_name,
  join_method,
  CASE
    WHEN campaign_id IS NOT NULL THEN 'Matched'
    ELSE 'Unavailable Mapping'
  END AS match_status

FROM joined
WHERE match_rank = 1;
```

---

## Output Columns

| Column | Source | Description |
|--------|--------|-------------|
| `mapped_channel` | Classification View | Dashboard Channel (e.g., Paid Social, Paid Search) |
| `mapped_partner` | Classification View | Dashboard Partner (e.g., Meta, Google Brand) |
| `mapped_campaign_focus` | Classification View | Prospecting, Retargeting, etc. |
| `campaign_grouping` | Mapping Table | NCO, Evergreen, Business Membership, etc. — or "Unavailable Mapping" |
| `campaign_type` | Mapping Table | Joins, Leads, Presale, etc. — or "Unavailable Mapping" |
| `region` | Mapping Table | AZ, CO, UT, Midwest — or "Unavailable Mapping" |
| `state` | Mapping Table | State breakdown — or "Unavailable Mapping" |
| `club_grouping` | Mapping Table | Seasoned, NCO, All Locations — or "Unavailable Mapping" |
| `mt_campaign_id` | Mapping Table | Platform campaign_id if matched, NULL if not |
| `join_method` | Bridge logic | How the match was made: Exact Campaign Match, Descriptor Match, or No Match |
| `match_status` | Bridge logic | "Matched" or "Unavailable Mapping" |

---

## QA Queries

### QA 1: Match Rate by Channel + Partner

Run this after deployment to see how well each channel is joining.

```sql
SELECT
  mapped_channel,
  mapped_partner,
  COUNT(*)                                                              AS total_rows,
  SUM(CASE WHEN match_status = 'Matched' THEN 1 ELSE 0 END)           AS matched_rows,
  SUM(CASE WHEN match_status = 'Unavailable Mapping' THEN 1 ELSE 0 END) AS unmatched_rows,
  ROUND(
    SUM(CASE WHEN match_status = 'Matched' THEN 1 ELSE 0 END)
    * 100.0 / COUNT(*), 1
  )                                                                     AS match_pct,
  SUM(total_joins)                                                      AS total_joins
FROM marketing.rb_to_platform_bridge
GROUP BY 1, 2
ORDER BY total_joins DESC;
```

**Target:** Paid Social and Search should be >60% at campaign level. Channel + Partner level should be 100%.

---

### QA 2: Match Method Distribution

See how many rows matched via exact name vs. descriptor vs. not at all.

```sql
SELECT
  join_method,
  COUNT(*)        AS rows,
  SUM(total_joins) AS total_joins,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS pct_of_rows
FROM marketing.rb_to_platform_bridge
GROUP BY 1
ORDER BY rows DESC;
```

---

### QA 3: Top Unavailable Mapping Rows by Joins

Find the highest-value unmatched rows to prioritize fixing.

```sql
SELECT
  mapped_channel,
  mapped_partner,
  tier_4,
  tier_4_normalized,
  campaign_descriptor,
  SUM(total_joins) AS total_joins
FROM marketing.rb_to_platform_bridge
WHERE match_status = 'Unavailable Mapping'
GROUP BY 1, 2, 3, 4, 5
ORDER BY total_joins DESC
LIMIT 50;
```

**Action:** Review the top 50. If a campaign is clearly mappable but the normalization missed it, add a rule or update the Mapping Table.

---

### QA 4: Duplicate Detection

Check if any RB row matched to multiple Mapping Table campaigns (the `match_rank = 1` filter should prevent this, but verify).

```sql
SELECT
  tier_1, tier_2, tier_3, tier_4, tier_5,
  COUNT(*) AS match_count
FROM marketing.rb_to_platform_bridge
GROUP BY 1, 2, 3, 4, 5
HAVING COUNT(*) > 1
ORDER BY match_count DESC
LIMIT 20;
```

**Expected:** 0 rows. If duplicates appear, the descriptor match is too broad — tighten the join condition.

---

### QA 5: Unavailable Mapping Trend (run weekly)

Track whether the unmapped rate is improving over time.

```sql
SELECT
  DATE_TRUNC('week', max_date) AS week,
  COUNT(*)                      AS total_rows,
  SUM(CASE WHEN match_status = 'Unavailable Mapping' THEN 1 ELSE 0 END) AS unmatched,
  ROUND(
    SUM(CASE WHEN match_status = 'Unavailable Mapping' THEN 1 ELSE 0 END)
    * 100.0 / COUNT(*), 1
  ) AS unmatched_pct
FROM marketing.rb_to_platform_bridge
WHERE mapped_channel IN ('Paid Social', 'Paid Search', 'Display', 'PMAX', 'Demand Generation')
GROUP BY 1
ORDER BY 1 DESC;
```

**Target:** Unmatched % should trend downward as the Mapping Table is updated with new campaigns. If it spikes, new campaigns are being launched without being added to the Mapping Table.

---

## Maintenance Notes

- **New campaigns:** When new campaigns launch, add them to `marketing.mapping_table_campaigns`. The bridge view will automatically pick them up on next refresh.
- **New Rockerbox tier values:** If RB introduces new tier_1 or tier_2 values, add mapping rules to the classification view.
- **Unavailable Mapping reviews:** Run QA 3 monthly. The top unmatched rows by joins are the highest priority to resolve — either by updating the Mapping Table or adding normalization rules.
