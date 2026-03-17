# Geopolitical Shocks & Energy Markets (2010–2026)

## Project Overview
This project examines how global geopolitical events affect global oil prices and how this impact is transmitted to to the Ukrainian market through exchange rate fluctuations.

The analysis focuses on two major oil benchmarks:
- Brent Crude Oil (global benchmark)
- West Texas Intermediate (WTI, US benchmark)

Additionally, the study incorporates currency dynamics between:
- United States Dollar
- Ukrainian Hryvnia

### Key Questions
- How do geopolitical events impact oil prices?
- Which events generate the strongest market reactions?
- Why does oil feel more expensive in Ukraine even when global prices remain stable?

## Tech Stack
- **SQL**: Google BigQuery (Standard SQL)
- **Visualization**: Looker Studio
- **Techniques Used**:
  - Window Functions (LAG, LEAD, AVG OVER)
  - CTEs
  - Joins
  - Aggregation & Time Series Analysis

## Data Sources
- Oil prices dataset (2010–2026)
- Geopolitical events dataset
- Exchange rate data (USD/UAH)

## 1. Baseline Market Dynamics
To understand long-term trends, yearly oil price statistics were calculated.
Metrics:
- Average price
- Minimum and maximum price
- Comparison between Brent and WTI

*SQL query:*
```sql
SELECT 
  EXTRACT(YEAR FROM date) AS year, 
  
  -- Brent statistics:
  ROUND(AVG(brent_price), 2) AS avg_brent,
  MIN(brent_price) AS min_brent,
  MAX(brent_price) AS max_brent,
  
  --WTI Crude Statistics:
  ROUND(AVG(wti_price), 2) AS avg_wti,
  MIN(wti_price) AS min_wti,
  MAX(wti_price) AS max_wti

FROM `oilpetproject.Data.oil_geopolitics_dataset_2010_2026`
GROUP BY year
ORDER BY year;
To analyze the distribution of geopolitical risks, I used the following query:


```
<img width="1363" height="818" alt="image" src="https://github.com/user-attachments/assets/92e74439-8a8a-4fbf-b9f6-6c66d316e65f" />

#### Key Findings
The Brent benchmark consistently shows higher sensitivity to global geopolitical events compared to WTI. This makes Brent a more reliable leading indicator for assessing energy risks and economic forecasting in the European and Ukrainian markets

## 2. Geopolitical Risk Analysis
Geopolitical events were categorized and analyzed by:
- Frequency
- Severity
- Type of shock
  
*SQL query:*
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

#### Key Findings
The analysis reveals a clear asymmetry between the frequency and impact of different event types.
High-impact events such as market crashes, oil price wars, and attacks  demonstrate the highest average severity but occur relatively rarely. Despite their low frequency, these events are responsible for the most extreme market movements.

In contrast, more frequent events  including sanctions, OPEC decisions, and regional conflicts  exhibit slightly lower average severity but contribute to sustained market pressure over time.

Notably, OPEC-related events combine both relatively high frequency and consistently strong impact, making them one of the most influential and predictable drivers of oil price dynamics

## 3. Oil Price Elasticity 
- Compare price before and after event
- Measure % change
- Identify most impactful event types

*SQL query:*
```sql
CREATE VIEW `oilpetproject.Data.v_oil_price_elasticity_analysis` AS
WITH market_trends AS (
  SELECT 
    main.date,
    main.brent_price,
    main.wti_price,
    ev.event_type,
    ev.event_description,
    -- Window functions to fetch future prices (7-day outlook)
    LEAD(main.brent_price, 7) OVER (ORDER BY main.date) as brent_after_7d,
    LEAD(main.wti_price, 7) OVER (ORDER BY main.date) as wti_after_7d,
    -- Initial price difference 
    (main.brent_price - main.wti_price) as current_spread
  FROM `oilpetproject.Data.oil_geopolitics_dataset_2010_2026` AS main
  LEFT JOIN `oilpetproject.Data.geopolitical_events_timeline` AS ev 
    ON main.date = ev.date
)

SELECT 
  date,
  event_type,
  event_description,
  -- Price Elasticity: % change 7 days post-event
  ROUND(((brent_after_7d - brent_price) / brent_price) * 100, 2) as brent_hike_7d_pct,
  ROUND(((wti_after_7d - wti_price) / wti_price) * 100, 2) as wti_hike_7d_pct,
  -- Reaction Gap: identifies which benchmark is more sensitive to the shock
  ROUND(
    ((brent_after_7d - brent_price) / brent_price) * 100 - 
    ((wti_after_7d - wti_price) / wti_price) * 100, 2
  ) as reaction_gap_pct,
  ROUND(current_spread, 2) as spread_at_event
FROM market_trends
WHERE event_type IS NOT NULL 
  AND brent_after_7d IS NOT NULL;
```
<img width="1477" height="707" alt="image" src="https://github.com/user-attachments/assets/1ec5a163-48f2-4659-912b-f8e1c45f8733" />

### Key Findings
Supply-related geopolitical events, particularly wars and OPEC decisions, generate the strongest positive price reactions. On average, oil prices increase by 15–28% within the first 7 days following such events.  markets respond primarily to anticipated supply disruptions rather than actual shortages.

Negative price movement was identified, with WTI showing a decline of approximately -140%. This anomaly does not reflect a typical market reaction to geopolitical events. Instead, it represents a technical market collapse in April 2020 caused by storage saturation during the COVID-19 pandemic. 

The observed decline exceeding -100% is a result of extreme market conditions combined with percentage change calculation methodology.
## 4. Impact of UAH Devaluation on Oil Prices in Ukraine
To understand how global oil price dynamics translate into the Ukrainian market, I combined Brent crude prices (USD) with the USD/UAH exchange rate.
The goal of this analysis is to evaluate whether Ukrainian consumers actually benefit from declines in global oil prices

 Methodology:

- Joined global oil price data with daily USD/UAH exchange rates
- Converted oil prices from USD to UAH
- Calculated daily price changes in UAH
- Applied a 30-day moving average to smooth short-term volatility

*SQL query:*
```sql
WITH oil_fx AS (
  SELECT
    o.date,
    o.gpr_index AS risk_idx,
    o.brent_price AS price_usd,
    e.USDUAH AS fx_rate,
    -- Currency conversion
    ROUND(o.brent_price * e.USDUAH, 2) AS price_uah
  FROM `oilpetproject.Data.oil_geopolitics_dataset_2010_2026` o
  JOIN `oilpetproject.Data.exchange_rate` e ON o.date = e.date
),

price_analysis AS (
  SELECT
    date,
    risk_idx,
    price_usd,
    fx_rate,
    price_uah,
    
    -- Daily price change (UAH)
    ROUND(price_uah - LAG(price_uah) OVER(ORDER BY date), 2) AS change_uah,

    -- 30-day moving averages
    ROUND(AVG(price_usd) OVER(ORDER BY date ROWS BETWEEN 30 PRECEDING AND CURRENT ROW), 2) AS avg_30d_usd,
    ROUND(AVG(price_uah) OVER(ORDER BY date ROWS BETWEEN 30 PRECEDING AND CURRENT ROW), 2) AS avg_30d_uah
  FROM oil_fx
)

SELECT *
FROM price_analysis
ORDER BY date DESC
```

<img width="1243" height="541" alt="image" src="https://github.com/user-attachments/assets/d4aaed8c-944e-49f5-9c14-8ed67f4b787a" />

The analysis shows that oil price dynamics in Ukraine are heavily influenced by currency fluctuations rather than global oil prices alone.
Even during periods of global price stability or decline (such as 2015 and 2020), domestic oil prices remained elevated or continued to grow due to the depreciation of the Ukrainian hryvnia (UAH).
This demonstrates that the **USD/UAH amplifies global price movements and, in some cases, completely offsets the benefits of declining oil prices.

### Conclusion

For the Ukrainian market, currency risk is a more significant driver of fuel costs than the nominal price of oil in USD. As a result, local consumers experience sustained price pressure even when global energy markets stabilize or decline.

