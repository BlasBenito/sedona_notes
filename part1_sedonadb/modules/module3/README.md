# Module 3: Advanced Spatial Queries
**Duration:** 2 hours

## Overview
Master complex spatial operations including joins, aggregations, window functions, and multi-step analyses.

---

## 3.1 Spatial Joins (30 minutes)

### Point-in-Polygon Join

```sql
-- Find which neighborhood each restaurant is in
SELECT
    r.name as restaurant,
    n.name as neighborhood
FROM restaurants r
JOIN neighborhoods n ON ST_Contains(n.geom, r.location);
```

### Distance Join

```sql
-- Find all parks within 1km of each school
SELECT
    s.name as school,
    p.name as park,
    ST_Distance(s.location, p.location) * 111 as distance_km
FROM schools s
JOIN parks p ON ST_DWithin(s.location, p.location, 0.01)
WHERE ST_Distance(s.location, p.location) * 111 < 1.0
ORDER BY s.name, distance_km;
```

### Polygon-Polygon Join

```sql
-- Find overlapping zoning districts
SELECT
    a.zone_id as zone_a,
    b.zone_id as zone_b,
    ST_Area(ST_Intersection(a.geom, b.geom)) as overlap_area
FROM zoning a
JOIN zoning b ON ST_Intersects(a.geom, b.geom)
WHERE a.zone_id < b.zone_id  -- Avoid duplicates
    AND ST_Area(ST_Intersection(a.geom, b.geom)) > 0;
```

---

## 3.2 Spatial Aggregations (25 minutes)

### Union Aggregation

```sql
-- Merge all parcels by owner
SELECT
    owner,
    ST_Union_Agg(geom) as combined_parcels,
    SUM(ST_Area(geom)) as total_area
FROM parcels
GROUP BY owner;
```

### Envelope Aggregation

```sql
-- Get bounding box of all points per category
SELECT
    category,
    ST_Envelope_Agg(location) as bbox,
    COUNT(*) as point_count
FROM locations
GROUP BY category;
```

### Statistical Aggregations with Spatial Filter

```sql
-- Average property value within each district
SELECT
    d.district_name,
    AVG(p.value) as avg_value,
    COUNT(p.id) as property_count,
    ST_Area(d.geom) as district_area
FROM districts d
LEFT JOIN properties p ON ST_Contains(d.geom, p.location)
GROUP BY d.district_name, d.geom
ORDER BY avg_value DESC;
```

---

## 3.3 K-Nearest Neighbors (20 minutes)

### Basic KNN Query

```sql
-- Find 5 nearest hospitals to each school
WITH school_hospitals AS (
    SELECT
        s.id as school_id,
        s.name as school,
        h.name as hospital,
        ST_Distance(s.location, h.location) * 111 as distance_km,
        ROW_NUMBER() OVER (
            PARTITION BY s.id
            ORDER BY ST_Distance(s.location, h.location)
        ) as rn
    FROM schools s
    CROSS JOIN hospitals h
)
SELECT school, hospital, distance_km
FROM school_hospitals
WHERE rn <= 5
ORDER BY school_id, rn;
```

### KNN with Additional Filters

```sql
-- Find nearest 24-hour pharmacy for each address
WITH ranked_pharmacies AS (
    SELECT
        a.address_id,
        p.name,
        p.hours,
        ST_Distance(a.location, p.location) as distance,
        ROW_NUMBER() OVER (
            PARTITION BY a.address_id
            ORDER BY ST_Distance(a.location, p.location)
        ) as rn
    FROM addresses a
    CROSS JOIN pharmacies p
    WHERE p.hours_24 = true
)
SELECT address_id, name, distance
FROM ranked_pharmacies
WHERE rn = 1;
```

---

## 3.4 Window Functions with Spatial Data (20 minutes)

```sql
-- Calculate cumulative coverage area
SELECT
    name,
    ST_Area(geom) as area,
    SUM(ST_Area(geom)) OVER (ORDER BY ST_Area(geom) DESC) as cumulative_area,
    SUM(ST_Area(geom)) OVER (ORDER BY ST_Area(geom) DESC) /
        SUM(ST_Area(geom)) OVER () * 100 as cumulative_pct
FROM districts
ORDER BY area DESC;

-- Rank locations by distance from center
SELECT
    name,
    ST_Distance(location, ST_Point(0, 0)) as distance_from_center,
    RANK() OVER (ORDER BY ST_Distance(location, ST_Point(0, 0))) as distance_rank,
    NTILE(4) OVER (ORDER BY ST_Distance(location, ST_Point(0, 0))) as quartile
FROM locations;
```

---

## 3.5 Complex Multi-Step Analysis (25 minutes)

### Example: Service Gap Analysis

```sql
-- Find underserved areas (high population, low service coverage)
WITH service_coverage AS (
    -- Calculate coverage from existing facilities
    SELECT
        c.census_block_id,
        c.population,
        c.geom as block_geom,
        COUNT(f.id) as num_facilities,
        MIN(ST_Distance(c.geom, f.location)) * 111 as nearest_facility_km
    FROM census_blocks c
    LEFT JOIN facilities f
        ON ST_DWithin(ST_Centroid(c.geom), f.location, 0.02)  -- 2km
    GROUP BY c.census_block_id, c.population, c.geom
),
ranked_blocks AS (
    SELECT
        *,
        NTILE(4) OVER (ORDER BY population DESC) as pop_quartile,
        NTILE(4) OVER (ORDER BY num_facilities) as service_quartile
    FROM service_coverage
)
SELECT
    census_block_id,
    population,
    num_facilities,
    nearest_facility_km,
    ST_AsText(ST_Centroid(block_geom)) as center
FROM ranked_blocks
WHERE pop_quartile = 1  -- High population
    AND service_quartile = 1  -- Low service
ORDER BY population DESC
LIMIT 20;
```

---

## Hands-on Exercise 3: Real Estate Analysis

```python
import duckdb

conn = duckdb.connect('real_estate.duckdb')
conn.execute("INSTALL spatial; LOAD spatial;")

# Create sample data
conn.execute("""
    CREATE TABLE properties AS
    SELECT
        row_number() OVER () as id,
        random() * 360 - 180 as lon,
        random() * 180 - 90 as lat,
        (random() * 500000 + 200000)::INTEGER as price,
        (random() * 3000 + 500)::INTEGER as sqft,
        ['SFH', 'Condo', 'Townhouse'][CAST(random() * 3 AS INTEGER) + 1] as type
    FROM range(10000)
""")

conn.execute("""
    ALTER TABLE properties
    ADD COLUMN location GEOMETRY
""")

conn.execute("""
    UPDATE properties
    SET location = ST_Point(lon, lat)
""")

# Analysis 1: Price per square foot by property type
result = conn.execute("""
    SELECT
        type,
        COUNT(*) as count,
        AVG(price) as avg_price,
        AVG(sqft) as avg_sqft,
        AVG(price::FLOAT / sqft) as avg_price_per_sqft
    FROM properties
    GROUP BY type
    ORDER BY avg_price_per_sqft DESC
""").df()

print("Price Analysis:")
print(result)

# Analysis 2: Find clusters of expensive properties
result = conn.execute("""
    WITH expensive_props AS (
        SELECT *
        FROM properties
        WHERE price > (SELECT PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY price) FROM properties)
    )
    SELECT
        p1.id,
        p1.price,
        COUNT(p2.id) as nearby_expensive_count
    FROM expensive_props p1
    LEFT JOIN expensive_props p2
        ON p1.id != p2.id
        AND ST_DWithin(p1.location, p2.location, 0.1)  -- ~10km
    GROUP BY p1.id, p1.price
    HAVING nearby_expensive_count >= 5
    ORDER BY nearby_expensive_count DESC
    LIMIT 10
""").df()

print("\nExpensive Property Clusters:")
print(result)
```

---

## Key Takeaways

- Spatial joins combine datasets based on geometric relationships
- Aggregations can compute union, envelope, and statistics
- KNN queries find nearest neighbors efficiently
- Window functions enable ranking and cumulative calculations
- Complex analyses combine multiple spatial operations
- CTEs (WITH clauses) break down complex queries into steps

---

**Previous:** [Module 2](../module2/README.md) | **Next:** [Module 4](../module4/README.md)
