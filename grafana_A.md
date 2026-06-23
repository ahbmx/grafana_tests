For an executive audience, I'd build **6–8 panels max**, with a strong emphasis on trends rather than current values. Executives usually care about:

* Are we growing?
* How fast are we growing?
* When will we run out?
* Which site is consuming storage faster?

## Recommended Layout

```
┌─────────────────────────────────────────────┐
│ Total Capacity │ Used │ Free │ Reduction    │
└─────────────────────────────────────────────┘

┌─────────────────────┬───────────────────────┐
│ Utilization by Site │ Capacity Remaining    │
└─────────────────────┴───────────────────────┘

┌─────────────────────────────────────────────┐
│ Physical Consumption Trend (30/90/180 days) │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Provisioned vs Physical Trend               │
└─────────────────────────────────────────────┘

┌─────────────────────┬───────────────────────┐
│ Growth Rate by Site │ Arrays Near Capacity  │
└─────────────────────┴───────────────────────┘
```

---

# Common CTE

Most KPI panels should use the latest snapshot per array:

```sql
WITH latest AS (
    SELECT DISTINCT ON (pure_array)
        pure_array,
        event_time,
        capacity,
        total_provisioned,
        total_physical,
        data_reduction
    FROM z_pure_capacity
    ORDER BY pure_array, event_time DESC
)
```

---

# Panel 1 - Total Capacity

```sql
WITH latest AS (
    SELECT DISTINCT ON (pure_array)
        pure_array,
        capacity
    FROM z_pure_capacity
    ORDER BY pure_array,event_time DESC
)
SELECT
ROUND(SUM(capacity)/1024.0,2) AS total_tb
FROM latest;
```

Grafana:

```json
{
  "type": "stat",
  "title": "Total Capacity (TB)"
}
```

---

# Panel 2 - Used Capacity

```sql
WITH latest AS (
    SELECT DISTINCT ON (pure_array)
        pure_array,
        total_physical
    FROM z_pure_capacity
    ORDER BY pure_array,event_time DESC
)
SELECT
ROUND(SUM(total_physical)/1024.0,2) AS used_tb
FROM latest;
```

---

# Panel 3 - Free Capacity

```sql
WITH latest AS (
    SELECT DISTINCT ON (pure_array)
        pure_array,
        capacity,
        total_physical
    FROM z_pure_capacity
    ORDER BY pure_array,event_time DESC
)
SELECT
ROUND(
SUM(capacity-total_physical)/1024.0,
2
) AS free_tb
FROM latest;
```

---

# Panel 4 - Average Data Reduction

```sql
WITH latest AS (
    SELECT DISTINCT ON (pure_array)
        pure_array,
        data_reduction
    FROM z_pure_capacity
    ORDER BY pure_array,event_time DESC
)
SELECT
ROUND(AVG(data_reduction),2)
FROM latest;
```

Unit:

```text
ratio
```

---

# Panel 5 - Utilization by Site

This should be a Bar Gauge.

```sql
WITH latest AS (
    SELECT DISTINCT ON (pure_array)
        pure_array,
        capacity,
        total_physical
    FROM z_pure_capacity
    ORDER BY pure_array,event_time DESC
)
SELECT
CASE
    WHEN pure_array LIKE 'MXTL%' THEN 'Toluca'
    WHEN pure_array LIKE 'MXCL%' THEN 'Colima'
END AS site,

ROUND(
SUM(total_physical)*100.0/
SUM(capacity),
2
) utilization
FROM latest
GROUP BY site;
```

Example JSON:

```json
{
  "type": "bargauge",
  "title": "Site Utilization %",
  "options": {
    "orientation": "horizontal"
  }
}
```

---

# Panel 6 - Physical Capacity Trend (Most Important)

Executives love this.

```sql
SELECT
$__timeGroupAlias(event_time,'1d'),
CASE
    WHEN pure_array LIKE 'MXTL%' THEN 'Toluca'
    WHEN pure_array LIKE 'MXCL%' THEN 'Colima'
END AS metric,
SUM(total_physical)/1024.0 AS used_tb
FROM z_pure_capacity
WHERE $__timeFilter(event_time)
GROUP BY 1,2
ORDER BY 1;
```

Visualization:

```json
{
  "type": "timeseries",
  "title": "Storage Consumption Growth"
}
```

You'll get:

```
Toluca  ───────────╮
                  ╰───
Colima  ──────╮
             ╰───────
```

This immediately shows growth trends.

---

# Panel 7 - Provisioned vs Physical

Very useful for explaining savings.

```sql
SELECT
$__timeGroupAlias(event_time,'1d'),

SUM(total_provisioned)/1024.0 AS provisioned_tb,

SUM(total_physical)/1024.0 AS physical_tb

FROM z_pure_capacity
WHERE $__timeFilter(event_time)
GROUP BY 1
ORDER BY 1;
```

Display:

```json
{
  "type": "timeseries",
  "title": "Provisioned vs Physical"
}
```

---

# Panel 8 - Monthly Growth Rate

Executives ask:

> How much are we growing per month?

```sql
WITH daily AS (
SELECT
date_trunc('day',event_time) d,
SUM(total_physical) physical
FROM z_pure_capacity
GROUP BY 1
)

SELECT
ROUND(
(
MAX(physical)-MIN(physical)
)
/1024.0
,2
) AS growth_tb
FROM daily
WHERE d > now() - interval '30 days';
```

Stat Panel.

Example:

```text
Growth Last 30 Days

+42 TB
```

---

# Capacity Forecast Query

For forecasting, feed Grafana a clean daily trend:

```sql
SELECT
$__timeGroupAlias(event_time,'1d'),
SUM(total_physical)/1024.0 AS used_tb
FROM z_pure_capacity
WHERE $__timeFilter(event_time)
GROUP BY 1
ORDER BY 1;
```

Then in Grafana 13:

1. Time Series panel
2. Transformations
3. Add Forecast
4. Forecast horizon = 180 days

This produces an executive-friendly chart showing projected capacity consumption.

---

# Dashboard JSON Skeleton

You can import this and then paste each query into the appropriate panel:

```json
{
  "title": "Pure Storage Executive Dashboard",
  "timezone": "browser",
  "schemaVersion": 41,
  "version": 1,
  "refresh": "5m",
  "panels": [
    {
      "id": 1,
      "title": "Total Capacity",
      "type": "stat",
      "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 }
    },
    {
      "id": 2,
      "title": "Used Capacity",
      "type": "stat",
      "gridPos": { "h": 4, "w": 6, "x": 6, "y": 0 }
    },
    {
      "id": 3,
      "title": "Free Capacity",
      "type": "stat",
      "gridPos": { "h": 4, "w": 6, "x": 12, "y": 0 }
    },
    {
      "id": 4,
      "title": "Data Reduction",
      "type": "stat",
      "gridPos": { "h": 4, "w": 6, "x": 18, "y": 0 }
    },
    {
      "id": 5,
      "title": "Site Utilization",
      "type": "bargauge",
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 4 }
    },
    {
      "id": 6,
      "title": "Storage Consumption Growth",
      "type": "timeseries",
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 4 }
    },
    {
      "id": 7,
      "title": "Provisioned vs Physical",
      "type": "timeseries",
      "gridPos": { "h": 8, "w": 24, "x": 0, "y": 12 }
    }
  ]
}
```

One thing I'd change for your environment: because you collect data every **5 minutes**, aggregate trend charts to **daily values** (`'1d'`) or even **weekly values** for executive dashboards. Showing 8,640 points per month adds noise; executives respond much better to smooth growth curves and forecasts.
