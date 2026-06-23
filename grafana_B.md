Since every row is a snapshot per array every 5 minutes, there's an important issue with the trend queries I gave earlier:

```sql
SUM(total_physical)
```

will **double-count** storage if multiple arrays report at the same timestamp, and it can create noisy growth charts.

For executive reporting, I'd build the trend panels from the **latest snapshot of each array per day**, then aggregate by site.

## Dashboard Variables

### Site Filter

Create a variable named `site`:

```sql
SELECT '__all' AS site
UNION
SELECT 'Toluca'
UNION
SELECT 'Colima'
ORDER BY 1;
```

Enable:

* Multi-value = Off
* Include All = On

---

# Panel 1: Total Capacity (TB)

```sql
WITH latest AS (
    SELECT DISTINCT ON (pure_array)
        pure_array,
        capacity
    FROM z_pure_capacity
    ORDER BY pure_array, event_time DESC
)
SELECT
    ROUND(SUM(capacity)/1024.0,2) AS "Capacity TB"
FROM latest
WHERE
(
    '${site}'='__all'
    OR (
        '${site}'='Toluca'
        AND pure_array LIKE 'MXTL%'
    )
    OR (
        '${site}'='Colima'
        AND pure_array LIKE 'MXCL%'
    )
);
```

---

# Panel 2: Used Capacity (TB)

```sql
WITH latest AS (
    SELECT DISTINCT ON (pure_array)
        pure_array,
        total_physical
    FROM z_pure_capacity
    ORDER BY pure_array, event_time DESC
)
SELECT
ROUND(SUM(total_physical)/1024.0,2)
AS "Used TB"
FROM latest
WHERE
(
    '${site}'='__all'
    OR (
        '${site}'='Toluca'
        AND pure_array LIKE 'MXTL%'
    )
    OR (
        '${site}'='Colima'
        AND pure_array LIKE 'MXCL%'
    )
);
```

---

# Panel 3: Utilization %

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
SUM(total_physical)*100.0/
SUM(capacity),
2
) AS utilization
FROM latest
WHERE
(
    '${site}'='__all'
    OR (
        '${site}'='Toluca'
        AND pure_array LIKE 'MXTL%'
    )
    OR (
        '${site}'='Colima'
        AND pure_array LIKE 'MXCL%'
    )
);
```

Gauge thresholds:

```text
0-70 Green
70-85 Yellow
85-100 Red
```

---

# Panel 4: Capacity Growth Trend

This is the chart executives will look at most.

```sql
WITH daily_snapshots AS (

    SELECT DISTINCT ON (
        date_trunc('day', event_time),
        pure_array
    )
        date_trunc('day', event_time) AS day,
        pure_array,
        total_physical

    FROM z_pure_capacity

    WHERE $__timeFilter(event_time)

    ORDER BY
        date_trunc('day', event_time),
        pure_array,
        event_time DESC
)

SELECT
    day AS time,
    SUM(total_physical)/1024.0 AS used_tb

FROM daily_snapshots

WHERE
(
    '${site}'='__all'
    OR (
        '${site}'='Toluca'
        AND pure_array LIKE 'MXTL%'
    )
    OR (
        '${site}'='Colima'
        AND pure_array LIKE 'MXCL%'
    )
)

GROUP BY day
ORDER BY day;
```

Visualization:

* Time Series
* Line width 3
* Fill opacity 15%

---

# Panel 5: Site Growth Comparison

This is excellent for leadership meetings.

```sql
WITH daily_snapshots AS (

    SELECT DISTINCT ON (
        date_trunc('day',event_time),
        pure_array
    )
        date_trunc('day',event_time) day,
        pure_array,
        total_physical

    FROM z_pure_capacity

    WHERE $__timeFilter(event_time)

    ORDER BY
        date_trunc('day',event_time),
        pure_array,
        event_time DESC
)

SELECT
    day AS time,

    CASE
        WHEN pure_array LIKE 'MXTL%' THEN 'Toluca'
        WHEN pure_array LIKE 'MXCL%' THEN 'Colima'
    END AS metric,

    SUM(total_physical)/1024.0 AS used_tb

FROM daily_snapshots

GROUP BY day, metric
ORDER BY day;
```

Produces:

```text
Toluca  ────────────────╮
                        │
                        ╰──────

Colima  ───────╮
               ╰────────────
```

---

# Panel 6: Provisioned vs Physical

Shows efficiency and overcommitment.

```sql
WITH daily_snapshots AS (

    SELECT DISTINCT ON (
        date_trunc('day',event_time),
        pure_array
    )
        date_trunc('day',event_time) day,
        pure_array,
        total_physical,
        total_provisioned

    FROM z_pure_capacity

    WHERE $__timeFilter(event_time)

    ORDER BY
        date_trunc('day',event_time),
        pure_array,
        event_time DESC
)

SELECT
    day AS time,
    SUM(total_provisioned)/1024.0 AS provisioned_tb,
    SUM(total_physical)/1024.0 AS physical_tb

FROM daily_snapshots

GROUP BY day
ORDER BY day;
```

---

# Panel 7: Growth Last 30 Days

A KPI executives immediately understand.

```sql
WITH daily AS (

    SELECT
        date_trunc('day',event_time) day,
        SUM(total_physical) physical

    FROM z_pure_capacity

    WHERE event_time >= now() - interval '30 days'

    GROUP BY 1
)

SELECT
ROUND(
(MAX(physical)-MIN(physical))/1024.0,
2
) AS growth_tb;
```

Example:

```text
+48 TB
Last 30 Days
```

---

# Panel 8: Days Until 80% Full

This becomes the favorite panel once it's working.

```sql
WITH daily AS (

    SELECT
        date_trunc('day',event_time) day,
        SUM(total_physical) physical

    FROM z_pure_capacity

    WHERE event_time >= now() - interval '60 days'

    GROUP BY 1
),

growth AS (

    SELECT
        (
            MAX(physical)-MIN(physical)
        ) /
        GREATEST(
            COUNT(*)-1,
            1
        ) AS daily_growth

    FROM daily
),

current_usage AS (

    SELECT
        SUM(total_physical) physical,
        SUM(capacity) capacity
    FROM (
        SELECT DISTINCT ON (pure_array)
            pure_array,
            total_physical,
            capacity
        FROM z_pure_capacity
        ORDER BY pure_array,event_time DESC
    ) x
)

SELECT
ROUND(
(
(current_usage.capacity*0.80)
-
current_usage.physical
)
/
growth.daily_growth
)
AS days_to_80pct

FROM growth,current_usage;
```

---

## What I would show to a CIO or Infrastructure Director

Top row:

* Total Capacity
* Used Capacity
* Utilization %
* Days Until 80%

Middle:

* Storage Consumption Growth (12 months)

Bottom:

* Toluca vs Colima Growth
* Provisioned vs Physical

That combination tells a complete story:

1. Current state
2. Growth trend
3. Efficiency
4. Forecasted exhaustion

It's the type of dashboard storage vendors and infrastructure executives typically use during quarterly capacity reviews.
