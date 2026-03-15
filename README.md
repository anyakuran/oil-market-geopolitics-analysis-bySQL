### Exploratory Data Analysis of oil prices and geopolitical risks (2010-2026) using SQL
##### This project analyzes the correlation between geopolitical events and Brent oil price volatility. The goal is to identify how different types of shocks (wars, sanctions, OPEC decisions) influence market prices.

#### Tech Stack
- Database: Google BigQuery (Standard SQL)
- BI Tool: Power BI / Looker Studio

### Key SQL Implementation

To analyze the distribution of geopolitical risks, I used the following query:

```sql
SELECT 
  event_type, 
  COUNT(*) as event_count,
  ROUND(AVG(event_severity), 2) as avg_severity,
  MIN(event_severity) as min_sev,
  MAX(event_severity) as max_sev
FROM oilpetproject.Data.geopolitical_events_timeline
GROUP BY event_type
ORDER BY event_count DESC;
```
<img width="1003" height="504" alt="image" src="https://github.com/user-attachments/assets/3bef5b74-b4e9-4243-ac2b-b4be9a1cd10e" />

```sql
WITH market_trends AS (
  SELECT 
    main.date,
    main.brent_price,
    main.wti_price,
    ev.event_type,
    ev.event_description,
    LEAD(main.brent_price, 7) OVER (ORDER BY main.date) as brent_after_7d,
    LEAD(main.wti_price, 7) OVER (ORDER BY main.date) as wti_after_7d,
    (main.brent_price - main.wti_price) as current_spread
  FROM `oilpetproject.Data.oil_geopolitics_dataset_2010_2026` AS main
  LEFT JOIN `oilpetproject.Data.geopolitical_events_timeline` AS ev 
    ON main.date = ev.date
)

SELECT 
  date,
  event_type,
  event_description,
  ROUND(((brent_after_7d - brent_price) / brent_price) * 100, 2) as brent_hike_7d_pct,
  ROUND(((wti_after_7d - wti_price) / wti_price) * 100, 2) as wti_hike_7d_pct,
  ROUND(
    ((brent_after_7d - brent_price) / brent_price) * 100 - 
    ((wti_after_7d - wti_price) / wti_price) * 100, 2
  ) as reaction_gap_pct,
  ROUND(current_spread, 2) as spread_at_event
FROM market_trends
WHERE event_type IS NOT NULL 
  AND brent_after_7d IS NOT NULL 
ORDER BY ABS(reaction_gap_pct) DESC; 
```


