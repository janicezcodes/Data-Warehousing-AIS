# Data-Warehousing-AIS

## Interactive Data Visualisation on Tableau
https://public.tableau.com/shared/3DXY2ZXJM?:display_count=n&:origin=viz_share_link

Here's an overview of the Dashboard 📈:
![Preview](./AIS%20Project_Data%20Visualisation.png)

## Project Summary

The automatic identification system (AIS) is a tracking system for ships. Ships use transceivers to broadcast, on very high-frequency radio, their identification, position, and speed, among other information, for other ships and stakeholders equipped with receivers to monitor them. This project uses AIS data from Data@Liánchéng (data.liancheng.science) which provides historical AIS data for part of Singapore waters.

A data warehouse was created and populated with these AIS messages using PostgreSQL. The star schema dimensional model was adopted in the data warehouse design. Four critical queries (listed below) were designed to extract insights into traffic patterns, vessel characteristics, port utilization and navigational safety. These insights are crucial for strategic planning, operational optimization, and risk mitigation in the Singapore Straits - one of the world's busiest maritime corridors. Finally, the above Tableau interface was built to query the data warehouse.

- Query 1: Vessels from which country and of what type contribute most to the
traffic in the Singapore Straits?
- Query 2: Which is the busiest grid in the Straits during day versus night?
- Query 3: Find out the number of vessels pulling over at Singapore’s ports.
- Query 4: Find out the MMSI of vessels whose destination is Singapore, and they
have a draught value greater than 15.7 meters (posing threats to navigational safety).

## SQL Queries

**1. Vessels from which country and of what type contribute most to the traffic in Singapore Straits?**

This query will aggregate the number of records in the fact_ais_type1 table by the flag and ship_type columns from the dim_vessel table, sorting the results to find the top contributor.

```sql
SELECT v.flag, v.ship_type, COUNT(*) as traffic_count
FROM fact_ais_type1 f1
JOIN dim_vessel v ON f1.mmsi = v.mmsi
GROUP BY v.flag, v.ship_type
ORDER BY traffic_count DESC;

--Result: 769 rows
```

**2. Which is the busiest grid in Day vs Night?**

This query will compare the counts of records in the fact_ais_type1 table during Day vs Night and AM vs PM, joined with the dim_time table for the time classification and dim_location table for grid identification.
> Note: This query assumes that the dim_time table has day_night and am_pm columns to indicate the time of day. The LIMIT 1 will give you the single busiest grid, but if you want the busiest per category (Day/Night, AM/PM), you would need to adjust the query accordingly.

```sql
WITH TrafficCounts AS (
    SELECT
        l.grid,
        t.am_pm,
        t.day_night,
        COUNT(*) as count
    FROM fact_ais_type1 f1
    JOIN dim_time t ON f1.hr = t.hr AND f1.min = t.min
    JOIN dim_location l ON f1.lon_deg = l.lon_deg AND f1.lon_min = l.lon_min AND f1.lat_deg = l.lat_deg AND f1.lat_min = l.lat_min
    GROUP BY l.grid, t.am_pm, t.day_night
)
SELECT
    grid,
    am_pm,
    day_night,
    MAX(count) as max_traffic
FROM TrafficCounts
GROUP BY grid, am_pm, day_night
ORDER BY max_traffic DESC;

--Result: (40,37) grid is the busiest grid in AM - Night time.
```

**3. Count the number of vessels with speed = 0 OR destination is Singapore?**

This query will count the number of unique vessels based on the conditions provided, joining the fact_ais_type1 and fact_ais_type5 tables with the respective dimension tables.

```sql
SELECT DISTINCT f.mmsi
FROM (
    SELECT mmsi FROM fact_ais_type1 WHERE speed = 0
    UNION
    SELECT mmsi FROM fact_ais_type5 f5
    JOIN dim_sg_destination d ON f5.destination = d.destination AND d.is_sg = 1
) AS f;

--Result: 9963 unique vessels with speed = 0 OR destination is Singapore, out of 25404 unique vessels in both fact tables.

-- edited:
(select DISTINCT f1.mmsi, 1 as datasource
from fact_ais_type1 f1
where f1.speed = 0)
union
(select DISTINCT f5.mmsi, 5 as datasource
from fact_ais_type5 f5, dim_sg_destination d
where d.is_sg = 1 and f5.destination = d.destination)
```

**4. Find MMSI of the vessels with destination = SG AND their draught > 15.7**

```sql
SELECT DISTINCT mmsi
FROM fact_ais_type5 f5
JOIN dim_sg_destination d ON f5.destination = d.destination
WHERE d.is_sg = 1 AND f5.draught > 15.7;

--Result: 234 unique vessels with destination = SG AND their draught > 15.7, out of 4878 unique vessels arriving in SG from fact table type 5, out of 7658 unique vessels from fact_ais_type5.
```