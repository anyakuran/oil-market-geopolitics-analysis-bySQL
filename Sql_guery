```sql
/*Geopolitical Shocks & Energy Markets (2010–2026)
SQL scripts for BigQuery analyzing oil prices vs geopolitical risks
===========================================================================
*/

-- 1. BASELINE MARKET DYNAMICS
-- Calculates annual average, min, and max prices for Brent and WTI benchmarks.
SELECT 
    EXTRACT(YEAR FROM date) AS year, 
    
    -- Brent statistics
    ROUND(AVG(brent_price), 2) AS avg_brent,
    MIN(brent_price) AS min_brent,
    MAX(brent_price) AS max_brent,
    
    -- WTI statistics
    ROUND(AVG(wti_price), 2) AS avg_wti,
    MIN(wti_price) AS min_wti,
    MAX(wti_price) AS max_wti
FROM `oilpetproject.Data.oil_geopolitics_dataset_2010_2026`
GROUP BY year
ORDER BY year;


-- 2. GEOPOLITICAL RISK ANALYSIS
-- Categorizes events by frequency and severity to identify high-impact shocks.
SELECT 
    event_type, 
    COUNT(*) as event_count,
    ROUND(AVG(event_severity), 2) as avg_severity,
    MIN(event_severity) as min_sev,
    MAX(event_severity) as max_sev
FROM `oilpetproject.Data.geopolitical_events_timeline`
GROUP BY event_type
ORDER BY event_count DESC;


-- 3. OIL PRICE ELASTICITY (VIEW CREATION)
-- Analyzes price reaction 7 days after a geopolitical event using window functions.
CREATE OR REPLACE VIEW `oilpetproject.Data.v_oil_price_elasticity_analysis` AS
WITH market_trends AS (
    SELECT 
        main.date,
        main.brent_price,
        main.wti_price,
        ev.event_type,
        ev.event_description,
        -- Fetch prices 7 days ahead to measure reaction
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
    -- Calculate % change 7 days post-event
    ROUND(((brent_after_7d - brent_price) / brent_price) * 100, 2) as brent_hike_7d_pct,
    ROUND(((wti_after_7d - wti_price) / wti_price) * 100, 2) as wti_hike_7d_pct,
    -- Sensitivity gap between benchmarks
    ROUND(
        ((brent_after_7d - brent_price) / brent_price) * 100 - 
        ((wti_after_7d - wti_price) / wti_price) * 100, 2
    ) as reaction_gap_pct,
    ROUND(current_spread, 2) as spread_at_event
FROM market_trends
WHERE event_type IS NOT NULL 
    AND brent_after_7d IS NOT NULL;


-- 4. IMPACT OF UAH DEVALUATION
-- Joins oil prices with FX rates and calculates 30-day moving averages.
WITH oil_fx AS (
    SELECT
        o.date,
        o.brent_price AS price_usd,
        e.USDUAH AS fx_rate,
        -- Convert to local currency (UAH)
        ROUND(o.brent_price * e.USDUAH, 2) AS price_uah
    FROM `oilpetproject.Data.oil_geopolitics_dataset_2010_2026` o
    JOIN `oilpetproject.Data.exchange_rate` e ON o.date = e.date
),
price_analysis AS (
    SELECT
        date,
        price_usd,
        fx_rate,
        price_uah,
        -- Daily volatility in UAH
        ROUND(price_uah - LAG(price_uah) OVER(ORDER BY date), 2) AS change_uah,
        -- Smoothing trends with Moving Averages
        ROUND(AVG(price_usd) OVER(ORDER BY date ROWS BETWEEN 30 PRECEDING AND CURRENT ROW), 2) AS avg_30d_usd,
        ROUND(AVG(price_uah) OVER(ORDER BY date ROWS BETWEEN 30 PRECEDING AND CURRENT ROW), 2) AS avg_30d_uah
    FROM oil_fx
)
SELECT *
FROM price_analysis
ORDER BY date DESC;
```

