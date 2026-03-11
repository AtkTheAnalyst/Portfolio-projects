# SQL Vehicle Theft Analysis 

This file presents the analysis from `SQL Vehicle Theft Analysis.sql` 

---

## Skills used

- Joins (including `LEFT JOIN`)
- Subqueries
- CTEs
- Aggregate functions
- `GROUP BY`, filtering, sorting
- Pivoting with `CASE WHEN`

---

## Setup and initial exploration

```sql
USE stolen_vehicles_db;
```

Preview the tables:

```sql
SELECT *
FROM locations;
```

```sql
SELECT *
FROM make_details;
```

```sql
SELECT *
FROM stolen_vehicles;
```

---

## Objective 1: Identify when vehicles are likely to be stolen

Goal: explore the date fields in `stolen_vehicles` to understand patterns over time.

### Vehicles stolen each year

```sql
SELECT COUNT(vehicle_id) AS num_of_vehicles,
       YEAR(date_stolen) AS yearr
FROM stolen_vehicles
GROUP BY yearr;
```

### Vehicles stolen each month

```sql
SELECT COUNT(vehicle_id) AS num_of_vehicles,
       MONTH(date_stolen) AS monthh
FROM stolen_vehicles
GROUP BY monthh;
```

### Month + year view (to check missing months)

```sql
SELECT COUNT(vehicle_id) AS num_of_vehicles,
       MONTH(date_stolen) AS monthh,
       YEAR(date_stolen) AS yearr
FROM stolen_vehicles
GROUP BY monthh, yearr
ORDER BY yearr, monthh;
```

### Check why April 2022 looks low (daily breakdown for month 4)

```sql
SELECT COUNT(vehicle_id) AS num_of_vehicles,
       MONTH(date_stolen) AS monthh,
       YEAR(date_stolen) AS yearr,
       DAY(date_stolen) AS dayy
FROM stolen_vehicles
WHERE MONTH(date_stolen) = 4
GROUP BY monthh, yearr, dayy
ORDER BY yearr, monthh, dayy;
```

### Vehicles stolen each day of the week (numeric)

```sql
SELECT COUNT(vehicle_id) AS num_of_vehicles,
       DAYOFWEEK(date_stolen) AS dayy
FROM stolen_vehicles
GROUP BY dayy
ORDER BY dayy;
```

### Vehicles stolen each day of the week (names)

```sql
SELECT COUNT(vehicle_id) AS num_of_vehicles,
       DAYNAME(date_stolen) AS day_of_week
FROM stolen_vehicles
GROUP BY day_of_week
ORDER BY day_of_week;
```

### Bar chart-friendly output (custom day ordering)

```sql
SELECT COUNT(vehicle_id) AS num_of_vehicles,
       DAYNAME(date_stolen) AS day_of_week_stolen
FROM stolen_vehicles
GROUP BY day_of_week_stolen
ORDER BY FIELD(day_of_week_stolen,
               'Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday');
```

Note: `FIELD()` is MySQL-specific ordering (useful for chart exports).

---

## Objective 2: Identify which vehicles are likely to be stolen

Goal: explore vehicle type, age, luxury vs standard, and color.

### Most commonly stolen vehicle types (top 5)

```sql
SELECT vehicle_type,
       COUNT(vehicle_id) AS num_of_vehicles
FROM stolen_vehicles
GROUP BY vehicle_type
ORDER BY num_of_vehicles DESC
LIMIT 5;
```

### Least commonly stolen vehicle types (bottom 5)

```sql
SELECT vehicle_type,
       COUNT(vehicle_id) AS num_of_vehicles
FROM stolen_vehicles
GROUP BY vehicle_type
ORDER BY num_of_vehicles ASC
LIMIT 5;
```

### Average age of stolen vehicles by type

```sql
SELECT vehicle_type,
       AVG(YEAR(date_stolen) - model_year) AS avg_age_of_cars_stolen
FROM stolen_vehicles
GROUP BY vehicle_type
ORDER BY avg_age_of_cars_stolen DESC;
```

### Percent luxury vs standard by vehicle type (CTE + CASE)

CTE preview:

```sql
WITH lux_standard AS (
  SELECT vehicle_type,
         CASE WHEN make_type = 'Luxury' THEN 1 ELSE 0 END AS luxury
  FROM stolen_vehicles sv
  LEFT JOIN make_details md ON sv.make_id = md.make_id
)
SELECT *
FROM lux_standard;
```

Final percent calculation:

```sql
WITH lux_standard AS (
  SELECT vehicle_type,
         CASE WHEN make_type = 'Luxury' THEN 1 ELSE 0 END AS luxury_or_standard
  FROM stolen_vehicles sv
  LEFT JOIN make_details md ON sv.make_id = md.make_id
)
SELECT vehicle_type,
       SUM(luxury_or_standard) / COUNT(*) * 100 AS percent_of_lux_vs_standard
FROM lux_standard
GROUP BY vehicle_type
ORDER BY percent_of_lux_vs_standard DESC;
```

### Color counts (to find top colors)

```sql
SELECT color,
       COUNT(vehicle_id) AS num_vehicles
FROM stolen_vehicles
GROUP BY color
ORDER BY num_vehicles DESC;
```

### Pivot table: top 10 vehicle types x top colors (plus “Other”)

```sql
SELECT vehicle_type,
       COUNT(vehicle_id) AS num_vehicles,
       SUM(CASE WHEN color = 'Silver' THEN 1 ELSE 0 END) AS silver,
       SUM(CASE WHEN color = 'White' THEN 1 ELSE 0 END) AS white,
       SUM(CASE WHEN color = 'Black' THEN 1 ELSE 0 END) AS black,
       SUM(CASE WHEN color = 'Blue' THEN 1 ELSE 0 END) AS blue,
       SUM(CASE WHEN color = 'Red' THEN 1 ELSE 0 END) AS red,
       SUM(CASE WHEN color = 'Grey' THEN 1 ELSE 0 END) AS Grey,
       SUM(CASE WHEN color = 'Green' THEN 1 ELSE 0 END) AS green,
       SUM(CASE WHEN color IN ('Gold', 'Brown', 'Yellow', 'Orange', 'Purple', 'Cream', 'Pink') THEN 1 ELSE 0 END) AS Other
FROM stolen_vehicles
GROUP BY vehicle_type
ORDER BY num_vehicles DESC
LIMIT 10;
```

Export note: this is suitable for a heatmap in Tableau/Excel.

---

## Objective 3: Identify where vehicles are likely to be stolen

Goal: combine theft counts with region population/density and prepare outputs for scatter plots and maps.

### Vehicles stolen by region

```sql
SELECT lo.region,
       COUNT(vehicle_id) AS num_of_vehicles_stolen
FROM stolen_vehicles sv
LEFT JOIN locations lo ON lo.location_id = sv.location_id
GROUP BY lo.region
ORDER BY num_of_vehicles_stolen DESC;
```

### Add population + density to the region output

```sql
SELECT lo.region,
       lo.population,
       lo.density,
       COUNT(vehicle_id) AS num_of_vehicles_stolen
FROM stolen_vehicles sv
LEFT JOIN locations lo ON lo.location_id = sv.location_id
GROUP BY lo.region, lo.population, lo.density
ORDER BY num_of_vehicles_stolen DESC;
```

### Compare vehicle types: high density vs low density regions (UNION)

```sql
(SELECT 'Low Density',
        sv.vehicle_type,
        COUNT(sv.vehicle_type) AS num_vehicles
 FROM stolen_vehicles sv
 LEFT JOIN locations lo ON lo.location_id = sv.location_id
 WHERE lo.region IN ('Otago', 'Gisborn', 'Southland')
 GROUP BY sv.vehicle_type
 ORDER BY num_vehicles DESC)
UNION
(SELECT 'High Density',
        sv.vehicle_type,
        COUNT(sv.vehicle_type) AS num_vehicles
 FROM stolen_vehicles sv
 LEFT JOIN locations lo ON lo.location_id = sv.location_id
 WHERE lo.region IN ('Auckland', 'Nelson', 'Wellington')
 GROUP BY sv.vehicle_type
 ORDER BY num_vehicles DESC);
```

Note: make sure region names match exactly (e.g. `Gisborne` vs `Gisborn`) in your dataset.

### Scatter plot export: population vs density

```sql
SELECT lo.population,
       lo.density
FROM stolen_vehicles sv
LEFT JOIN locations lo ON lo.location_id = sv.location_id;
```

### Map export: region-level counts

```sql
SELECT lo.region,
       sv.vehicle_id
FROM stolen_vehicles sv
LEFT JOIN locations lo ON lo.location_id = sv.location_id
GROUP BY sv.vehicle_id, lo.region;
```

```sql
SELECT lo.region,
       COUNT(sv.vehicle_id) AS num_stolen_vehicles
FROM stolen_vehicles sv
LEFT JOIN locations lo ON lo.location_id = sv.location_id
GROUP BY lo.region;
```

