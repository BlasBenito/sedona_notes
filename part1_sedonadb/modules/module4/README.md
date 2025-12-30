# Module 4: Data Integration and ETL
**Duration:** 1.5 hours

## 4.1 Multi-Format Data Loading (30 min)

```python
import duckdb
conn = duckdb.connect('etl.duckdb')
conn.execute("INSTALL spatial; LOAD spatial;")

# CSV with coordinates
conn.execute("""
    CREATE TABLE from_csv AS
    SELECT *, ST_Point(CAST(lon AS DOUBLE), CAST(lat AS DOUBLE)) as geom
    FROM read_csv('data.csv', AUTO_DETECT=TRUE)
""")

# GeoJSON
conn.execute("CREATE TABLE from_geojson AS SELECT * FROM ST_Read('data.geojson')")

# Shapefile
conn.execute("CREATE TABLE from_shp AS SELECT * FROM ST_Read('boundaries.shp')")

# GeoParquet (recommended for performance)
conn.execute("CREATE TABLE from_parquet AS SELECT * FROM read_parquet('data.geoparquet')")

# PostGIS via DuckDB postgres scanner
conn.execute("INSTALL postgres; LOAD postgres;")
conn.execute("""
    ATTACH 'dbname=mydb user=user host=localhost' AS pg (TYPE POSTGRES);
    CREATE TABLE from_postgis AS
    SELECT *, ST_GeomFromWKB(geom) as geometry
    FROM pg.spatial_table;
""")
```

## 4.2 Data Export (15 min)

```python
# Export to GeoJSON
conn.execute("""
    COPY (
        SELECT ST_AsGeoJSON(geom) as geometry, *
        FROM spatial_table
    ) TO 'output.geojson'
""")

# Export to GeoParquet (best for reuse)
conn.execute("""
    COPY spatial_table TO 'output.geoparquet' (FORMAT PARQUET)
""")

# Export to CSV with WKT
conn.execute("""
    COPY (
        SELECT ST_AsText(geom) as wkt, *
        FROM spatial_table
    ) TO 'output.csv' (HEADER, DELIMITER ',')
""")
```

## 4.3 Data Validation and Cleaning (25 min)

```sql
-- Check for invalid geometries
SELECT COUNT(*) FROM spatial_table WHERE NOT ST_IsValid(geom);

-- Fix invalid geometries
UPDATE spatial_table SET geom = ST_MakeValid(geom) WHERE NOT ST_IsValid(geom);

-- Remove duplicates
DELETE FROM spatial_table
WHERE id NOT IN (
    SELECT MIN(id)
    FROM spatial_table
    GROUP BY ST_AsText(geom)
);

-- Check for nulls
SELECT COUNT(*) FROM spatial_table WHERE geom IS NULL;

-- Validate CRS
SELECT DISTINCT ST_SRID(geom) FROM spatial_table;
```

## 4.4 Schema Design (20 min)

```sql
-- Good spatial table design
CREATE TABLE optimized_spatial (
    id INTEGER PRIMARY KEY,
    name VARCHAR NOT NULL,
    category VARCHAR,
    value DOUBLE,
    geom GEOMETRY NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Add spatial index
CREATE INDEX idx_geom ON optimized_spatial USING RTREE (geom);

-- Add attribute indexes
CREATE INDEX idx_category ON optimized_spatial(category);
CREATE INDEX idx_value ON optimized_spatial(value);
```

---

**Previous:** [Module 3](../module3/README.md) | **Next:** [Module 5](../module5/README.md)
