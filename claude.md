Here's a complete Grafana dashboard JSON you can import directly, covering both **Capacity** and **Health** executive views — built for upper management with clean KPIs, trend lines, and at-a-glance status panels.

Let me first show you a preview of the layout concept:Now here's the full Grafana dashboard JSON. It's structured with two rows — Capacity and Health — and uses PostgreSQL queries directly against your tables.Here's everything included and how to import it.

**To import:** In Grafana go to Dashboards → Import → Upload JSON file. On the next screen it will ask you to map `DS_POSTGRESQL` to your existing PostgreSQL datasource.

**What's in the dashboard:**

The Capacity section has six KPI stat panels across the top (total raw TB, used TB, free TB, overall utilization gauge, avg data reduction ratio, avg subscription %), three horizontal bar gauge panels breaking down utilization by location / domain / family, a 30-day used TB trend line split by location, and a table that surfaces only devices at ≥70% — color-coded orange/red.

The Health section mirrors that structure: six stat cards (fleet avg score, devices monitored, online components, offline components, critical count, degraded count), a full device health table with the `health_score` column rendered as a color-coded background so red/orange/green jumps out immediately, a 30-day trend line for the fleet avg score with threshold bands, and three bar gauges breaking down avg health score by domain / location, plus component availability % by family.

**Template variables** at the top let execs filter by Location and Domain without touching queries.

**A few things to verify before using it:**

- The queries use `WHERE location ~ '$location'` (PostgreSQL regex). If your Grafana variable multi-select format differs, you may need to switch to `WHERE location = ANY(ARRAY[$location])` or use `{location:csv}` format — depends on your Grafana version.
- Health score thresholds (90 = green, 80 = orange, <80 = red) and capacity thresholds (85% = red, 70% = orange) are set inline — adjust them in the field config overrides to match your SLAs.
- The trend queries aggregate by `date_trunc('day', event_time)` — if your `event_time` resolution is already daily, that's fine; if it's hourly you might want `'hour'` for shorter time windows.


```json
{
  "title": "Storage Executive Overview",
  "uid": "storage-exec-v1",
  "schemaVersion": 38,
  "version": 1,
  "refresh": "5m",
  "time": { "from": "now-30d", "to": "now" },
  "timepicker": {},
  "tags": ["storage", "executive", "capacity", "health"],
  "templating": {
    "list": [
      {
        "name": "location",
        "label": "Location",
        "type": "query",
        "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
        "query": "SELECT DISTINCT location FROM report_capacity ORDER BY location",
        "refresh": 2,
        "includeAll": true,
        "allValue": ".*",
        "multi": true,
        "current": {}
      },
      {
        "name": "domain",
        "label": "Domain",
        "type": "query",
        "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
        "query": "SELECT DISTINCT domain FROM report_capacity ORDER BY domain",
        "refresh": 2,
        "includeAll": true,
        "allValue": ".*",
        "multi": true,
        "current": {}
      }
    ]
  },
  "panels": [

    {
      "id": 1,
      "type": "text",
      "title": "",
      "gridPos": { "x": 0, "y": 0, "w": 24, "h": 1 },
      "options": {
        "mode": "markdown",
        "content": "## Capacity"
      }
    },

    {
      "id": 10,
      "type": "stat",
      "title": "Total Raw Capacity",
      "description": "Sum of total_tb across all systems in the selected time window",
      "gridPos": { "x": 0, "y": 1, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, ROUND(SUM(total_tb)::numeric, 2) AS \"Total TB\" FROM report_capacity WHERE event_time = (SELECT MAX(event_time) FROM report_capacity) AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "orientation": "auto",
        "textMode": "auto",
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center",
        "unit": "TB"
      },
      "fieldConfig": {
        "defaults": {
          "unit": "none",
          "decimals": 1,
          "displayName": "Total Raw (TB)",
          "color": { "mode": "fixed", "fixedColor": "blue" }
        }
      }
    },

    {
      "id": 11,
      "type": "stat",
      "title": "Used Capacity",
      "gridPos": { "x": 4, "y": 1, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, ROUND(SUM(used_tb)::numeric, 2) AS \"Used TB\" FROM report_capacity WHERE event_time = (SELECT MAX(event_time) FROM report_capacity) AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "none",
          "decimals": 1,
          "displayName": "Used (TB)",
          "color": { "mode": "fixed", "fixedColor": "orange" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center"
      }
    },

    {
      "id": 12,
      "type": "stat",
      "title": "Free Capacity",
      "gridPos": { "x": 8, "y": 1, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, ROUND(SUM(free_tb)::numeric, 2) AS \"Free TB\" FROM report_capacity WHERE event_time = (SELECT MAX(event_time) FROM report_capacity) AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "none",
          "decimals": 1,
          "displayName": "Free (TB)",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "orange", "value": 50 },
              { "color": "green", "value": 200 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center"
      }
    },

    {
      "id": 13,
      "type": "gauge",
      "title": "Overall Utilization %",
      "gridPos": { "x": 12, "y": 1, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, ROUND((SUM(used_tb)/NULLIF(SUM(total_tb),0)*100)::numeric, 1) AS \"Used %\" FROM report_capacity WHERE event_time = (SELECT MAX(event_time) FROM report_capacity) AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "max": 100,
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 70 },
              { "color": "red", "value": 85 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "showThresholdLabels": true,
        "showThresholdMarkers": true
      }
    },

    {
      "id": 14,
      "type": "stat",
      "title": "Avg Data Reduction Ratio",
      "gridPos": { "x": 16, "y": 1, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, ROUND(AVG(data_reduction)::numeric, 2) AS \"Avg Reduction\" FROM report_capacity WHERE event_time = (SELECT MAX(event_time) FROM report_capacity) AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "none",
          "decimals": 2,
          "displayName": "Reduction Ratio",
          "color": { "mode": "fixed", "fixedColor": "teal" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center"
      }
    },

    {
      "id": 15,
      "type": "stat",
      "title": "Avg Subscription %",
      "gridPos": { "x": 20, "y": 1, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, ROUND(AVG(subs_perc)::numeric, 1) AS \"Subscription %\" FROM report_capacity WHERE event_time = (SELECT MAX(event_time) FROM report_capacity) AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 80 },
              { "color": "red", "value": 100 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center"
      }
    },

    {
      "id": 20,
      "type": "bargauge",
      "title": "Utilization % by Location (latest snapshot)",
      "gridPos": { "x": 0, "y": 5, "w": 8, "h": 8 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT location, ROUND((SUM(used_tb)/NULLIF(SUM(total_tb),0)*100)::numeric,1) AS used_pct FROM report_capacity WHERE event_time = (SELECT MAX(event_time) FROM report_capacity) AND location ~ '$location' AND domain ~ '$domain' GROUP BY location ORDER BY used_pct DESC",
        "format": "table"
      }],
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "max": 100,
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 70 },
              { "color": "red", "value": 85 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "orientation": "horizontal",
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "displayMode": "gradient",
        "showUnfilled": true,
        "valueMode": "color"
      }
    },

    {
      "id": 21,
      "type": "bargauge",
      "title": "Utilization % by Domain (latest snapshot)",
      "gridPos": { "x": 8, "y": 5, "w": 8, "h": 8 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT domain, ROUND((SUM(used_tb)/NULLIF(SUM(total_tb),0)*100)::numeric,1) AS used_pct FROM report_capacity WHERE event_time = (SELECT MAX(event_time) FROM report_capacity) AND location ~ '$location' AND domain ~ '$domain' GROUP BY domain ORDER BY used_pct DESC",
        "format": "table"
      }],
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "max": 100,
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 70 },
              { "color": "red", "value": 85 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "orientation": "horizontal",
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "displayMode": "gradient",
        "showUnfilled": true,
        "valueMode": "color"
      }
    },

    {
      "id": 22,
      "type": "bargauge",
      "title": "Utilization % by Storage Family",
      "gridPos": { "x": 16, "y": 5, "w": 8, "h": 8 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT family, ROUND((SUM(used_tb)/NULLIF(SUM(total_tb),0)*100)::numeric,1) AS used_pct FROM report_capacity WHERE event_time = (SELECT MAX(event_time) FROM report_capacity) AND location ~ '$location' AND domain ~ '$domain' GROUP BY family ORDER BY used_pct DESC",
        "format": "table"
      }],
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "max": 100,
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 70 },
              { "color": "red", "value": 85 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "orientation": "horizontal",
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "displayMode": "gradient",
        "showUnfilled": true,
        "valueMode": "color"
      }
    },

    {
      "id": 23,
      "type": "timeseries",
      "title": "Total Used TB — Trend (30 days)",
      "gridPos": { "x": 0, "y": 13, "w": 16, "h": 8 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT date_trunc('day', event_time) AS time, location, ROUND(SUM(used_tb)::numeric, 2) AS used_tb FROM report_capacity WHERE event_time >= NOW() - INTERVAL '30 days' AND location ~ '$location' AND domain ~ '$domain' GROUP BY 1, 2 ORDER BY 1",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "none",
          "custom": { "lineWidth": 2, "fillOpacity": 10 }
        }
      },
      "options": {
        "tooltip": { "mode": "multi" },
        "legend": { "displayMode": "list", "placement": "bottom" }
      }
    },

    {
      "id": 24,
      "type": "table",
      "title": "Devices Nearing Capacity (used_perc ≥ 70%)",
      "gridPos": { "x": 16, "y": 13, "w": 8, "h": 8 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT device, vendor, location, domain, family, ROUND(total_tb::numeric,1) AS total_tb, ROUND(used_tb::numeric,1) AS used_tb, ROUND(used_perc::numeric,1) AS \"used_%\" FROM report_capacity WHERE event_time = (SELECT MAX(event_time) FROM report_capacity) AND used_perc >= 70 AND location ~ '$location' AND domain ~ '$domain' ORDER BY used_perc DESC",
        "format": "table"
      }],
      "options": { "sortBy": [{ "displayName": "used_%", "desc": true }] },
      "fieldConfig": {
        "overrides": [{
          "matcher": { "id": "byName", "options": "used_%" },
          "properties": [{
            "id": "custom.displayMode",
            "value": "color-background"
          }, {
            "id": "thresholds",
            "value": {
              "mode": "absolute",
              "steps": [
                { "color": "orange", "value": null },
                { "color": "red", "value": 85 }
              ]
            }
          }, {
            "id": "color",
            "value": { "mode": "thresholds" }
          }]
        }]
      }
    },

    {
      "id": 2,
      "type": "text",
      "title": "",
      "gridPos": { "x": 0, "y": 21, "w": 24, "h": 1 },
      "options": {
        "mode": "markdown",
        "content": "## Health"
      }
    },

    {
      "id": 30,
      "type": "stat",
      "title": "Fleet Avg Health Score",
      "gridPos": { "x": 0, "y": 22, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, ROUND(AVG(health_score)::numeric, 1) AS \"Health Score\" FROM report_health_summary WHERE event_time = (SELECT MAX(event_time) FROM report_health_summary) AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "none",
          "decimals": 1,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "orange", "value": 80 },
              { "color": "green", "value": 90 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center"
      }
    },

    {
      "id": 31,
      "type": "stat",
      "title": "Devices Monitored",
      "gridPos": { "x": 4, "y": 22, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, COUNT(DISTINCT device) AS \"Devices\" FROM report_health_summary WHERE event_time = (SELECT MAX(event_time) FROM report_health_summary) AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "fixed", "fixedColor": "blue" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center"
      }
    },

    {
      "id": 32,
      "type": "stat",
      "title": "Online Components",
      "gridPos": { "x": 8, "y": 22, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, SUM(online_components) AS \"Online\", SUM(total_components) AS \"Total\" FROM report_health_summary WHERE event_time = (SELECT MAX(event_time) FROM report_health_summary) AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "fixed", "fixedColor": "green" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center"
      }
    },

    {
      "id": 33,
      "type": "stat",
      "title": "Components Offline",
      "gridPos": { "x": 12, "y": 22, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, SUM(total_components - online_components) AS \"Offline\" FROM report_health_summary WHERE event_time = (SELECT MAX(event_time) FROM report_health_summary) AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 1 },
              { "color": "red", "value": 10 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center"
      }
    },

    {
      "id": 34,
      "type": "stat",
      "title": "Critical Devices (score < 80)",
      "gridPos": { "x": 16, "y": 22, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, COUNT(*) AS \"Critical\" FROM report_health_summary WHERE event_time = (SELECT MAX(event_time) FROM report_health_summary) AND health_score < 80 AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 1 },
              { "color": "red", "value": 5 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center"
      }
    },

    {
      "id": 35,
      "type": "stat",
      "title": "Degraded Devices (score 80–89)",
      "gridPos": { "x": 20, "y": 22, "w": 4, "h": 4 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT NOW() AS time, COUNT(*) AS \"Degraded\" FROM report_health_summary WHERE event_time = (SELECT MAX(event_time) FROM report_health_summary) AND health_score >= 80 AND health_score < 90 AND location ~ '$location' AND domain ~ '$domain'",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 1 },
              { "color": "red", "value": 10 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "none",
        "justifyMode": "center"
      }
    },

    {
      "id": 40,
      "type": "table",
      "title": "Device Health Summary (all devices, latest snapshot)",
      "gridPos": { "x": 0, "y": 26, "w": 16, "h": 10 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT device, domain, role, family, location, total_components, online_components, (total_components - online_components) AS offline_components, ROUND(health_score::numeric, 1) AS health_score FROM report_health_summary WHERE event_time = (SELECT MAX(event_time) FROM report_health_summary) AND location ~ '$location' AND domain ~ '$domain' ORDER BY health_score ASC",
        "format": "table"
      }],
      "options": { "sortBy": [{ "displayName": "health_score", "desc": false }] },
      "fieldConfig": {
        "overrides": [
          {
            "matcher": { "id": "byName", "options": "health_score" },
            "properties": [
              { "id": "custom.displayMode", "value": "color-background" },
              {
                "id": "thresholds",
                "value": {
                  "mode": "absolute",
                  "steps": [
                    { "color": "red", "value": null },
                    { "color": "orange", "value": 80 },
                    { "color": "green", "value": 90 }
                  ]
                }
              },
              { "id": "color", "value": { "mode": "thresholds" } }
            ]
          },
          {
            "matcher": { "id": "byName", "options": "offline_components" },
            "properties": [
              { "id": "custom.displayMode", "value": "color-text" },
              {
                "id": "thresholds",
                "value": {
                  "mode": "absolute",
                  "steps": [
                    { "color": "green", "value": null },
                    { "color": "orange", "value": 1 },
                    { "color": "red", "value": 5 }
                  ]
                }
              },
              { "id": "color", "value": { "mode": "thresholds" } }
            ]
          }
        ]
      }
    },

    {
      "id": 41,
      "type": "timeseries",
      "title": "Fleet Avg Health Score — Trend (30 days)",
      "gridPos": { "x": 16, "y": 26, "w": 8, "h": 10 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT date_trunc('day', event_time) AS time, ROUND(AVG(health_score)::numeric, 2) AS \"Avg Health Score\" FROM report_health_summary WHERE event_time >= NOW() - INTERVAL '30 days' AND location ~ '$location' AND domain ~ '$domain' GROUP BY 1 ORDER BY 1",
        "format": "time_series"
      }],
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "max": 100,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "orange", "value": 80 },
              { "color": "green", "value": 90 }
            ]
          },
          "color": { "mode": "thresholds" },
          "custom": { "lineWidth": 2, "fillOpacity": 15, "thresholdsStyle": { "mode": "area" } }
        }
      },
      "options": {
        "tooltip": { "mode": "single" },
        "legend": { "displayMode": "list", "placement": "bottom" }
      }
    },

    {
      "id": 42,
      "type": "bargauge",
      "title": "Avg Health Score by Domain",
      "gridPos": { "x": 0, "y": 36, "w": 8, "h": 6 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT domain, ROUND(AVG(health_score)::numeric, 1) AS health_score FROM report_health_summary WHERE event_time = (SELECT MAX(event_time) FROM report_health_summary) AND location ~ '$location' AND domain ~ '$domain' GROUP BY domain ORDER BY health_score ASC",
        "format": "table"
      }],
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "max": 100,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "orange", "value": 80 },
              { "color": "green", "value": 90 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "orientation": "horizontal",
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "displayMode": "gradient",
        "showUnfilled": true
      }
    },

    {
      "id": 43,
      "type": "bargauge",
      "title": "Avg Health Score by Location",
      "gridPos": { "x": 8, "y": 36, "w": 8, "h": 6 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT location, ROUND(AVG(health_score)::numeric, 1) AS health_score FROM report_health_summary WHERE event_time = (SELECT MAX(event_time) FROM report_health_summary) AND location ~ '$location' AND domain ~ '$domain' GROUP BY location ORDER BY health_score ASC",
        "format": "table"
      }],
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "max": 100,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "orange", "value": 80 },
              { "color": "green", "value": 90 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "orientation": "horizontal",
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "displayMode": "gradient",
        "showUnfilled": true
      }
    },

    {
      "id": 44,
      "type": "bargauge",
      "title": "Component Availability % by Family",
      "gridPos": { "x": 16, "y": 36, "w": 8, "h": 6 },
      "datasource": { "type": "postgres", "uid": "${DS_POSTGRESQL}" },
      "targets": [{
        "rawSql": "SELECT family, ROUND((SUM(online_components)::numeric / NULLIF(SUM(total_components),0) * 100), 1) AS availability_pct FROM report_health_summary WHERE event_time = (SELECT MAX(event_time) FROM report_health_summary) AND location ~ '$location' AND domain ~ '$domain' GROUP BY family ORDER BY availability_pct ASC",
        "format": "table"
      }],
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "max": 100,
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "orange", "value": 95 },
              { "color": "green", "value": 99 }
            ]
          },
          "color": { "mode": "thresholds" }
        }
      },
      "options": {
        "orientation": "horizontal",
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "displayMode": "gradient",
        "showUnfilled": true
      }
    }

  ],

  "__inputs": [
    {
      "name": "DS_POSTGRESQL",
      "label": "PostgreSQL",
      "description": "Your PostgreSQL datasource",
      "type": "datasource",
      "pluginId": "postgres",
      "pluginName": "PostgreSQL"
    }
  ],
  "__requires": [
    { "type": "grafana", "id": "grafana", "name": "Grafana", "version": "10.0.0" },
    { "type": "datasource", "id": "postgres", "name": "PostgreSQL", "version": "1.0.0" },
    { "type": "panel", "id": "stat", "name": "Stat", "version": "" },
    { "type": "panel", "id": "gauge", "name": "Gauge", "version": "" },
    { "type": "panel", "id": "bargauge", "name": "Bar gauge", "version": "" },
    { "type": "panel", "id": "timeseries", "name": "Time series", "version": "" },
    { "type": "panel", "id": "table", "name": "Table", "version": "" },
    { "type": "panel", "id": "text", "name": "Text", "version": "" }
  ]
}
```
