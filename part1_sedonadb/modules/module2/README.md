# Module 2: Spatial SQL Fundamentals
**Duration:** 1.5 hours

## Overview
Master the essential spatial SQL operations in SedonaDB, from creating geometries to performing measurements and transformations.

---

## 2.1 Geometry Types (15 minutes)

### OGC Simple Features

```sql
-- Point
SELECT ST_AsText(ST_Point(0, 0));
-- Result: POINT (0 0)

-- LineString
SELECT ST_AsText(ST_GeomFromText('LINESTRING (0 0, 1 1, 2 0)'));

-- Polygon
SELECT ST_AsText(ST_GeomFromText('POLYGON ((0 0, 4 0, 4 4, 0 4, 0 0))'));

-- MultiPoint
SELECT ST_AsText(ST_GeomFromText('MULTIPOINT ((0 0), (1 1), (2 2))'));

-- MultiPolygon
SELECT ST_AsText(ST_GeomFromText('MULTIPOLYGON (((0 0, 1 0, 1 1, 0 1, 0 0)), ((2 2, 3 2, 3 3, 2 3, 2 2)))'));
```

---

## 2.2 Loading Spatial Data (20 minutes)

### From CSV with Coordinates

```python
import duckdb

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

# Load CSV and create geometries
conn.execute("""
    CREATE TABLE locations AS
    SELECT
        *,
        ST_Point(lon, lat) as geom
    FROM read_csv('locations.csv', AUTO_DETECT=TRUE)
""")
```

### From GeoJSON

```python
# Load GeoJSON file
conn.execute("""
    CREATE TABLE buildings AS
    SELECT * FROM ST_Read('buildings.geojson')
""")
```

### From Shapefile

```python
# Load Shapefile
conn.execute("""
    CREATE TABLE boundaries AS
    SELECT * FROM ST_Read('boundaries.shp')
""")
```

### From GeoParquet

```python
# Most efficient format
conn.execute("""
    CREATE TABLE spatial_data AS
    SELECT * FROM read_parquet('data.geoparquet')
""")
```

---

## 2.3 Coordinate Reference Systems (15 minutes)

### Setting and Transforming CRS

```sql
-- Set SRID (Spatial Reference ID)
SELECT ST_SetSRID(ST_Point(-74.006, 40.7128), 4326) as geom_wgs84;

-- Transform CRS (WGS84 to Web Mercator)
SELECT ST_Transform(
    ST_SetSRID(ST_Point(-74.006, 40.7128), 4326),
    3857
) as geom_web_mercator;
```

### Common SRID Values

- 4326: WGS84 (GPS coordinates)
- 3857: Web Mercator (web maps)
- 2163: US National Atlas Equal Area

---

## 2.4 Spatial Predicates (20 minutes)

```sql
-- Contains
SELECT ST_Contains(
    ST_GeomFromText('POLYGON ((0 0, 4 0, 4 4, 0 4, 0 0))'),
    ST_Point(2, 2)
); -- true

-- Intersects
SELECT ST_Intersects(
    ST_GeomFromText('LINESTRING (0 0, 2 2)'),
    ST_GeomFromText('LINESTRING (0 2, 2 0)')
); -- true

-- Within
SELECT ST_Within(
    ST_Point(1, 1),
    ST_GeomFromText('POLYGON ((0 0, 3 0, 3 3, 0 3, 0 0))')
); -- true

-- DWithin (distance query)
SELECT ST_DWithin(
    ST_Point(0, 0),
    ST_Point(3, 4),
    10
); -- true (distance is 5)
```

---

## 2.5 Spatial Measurements (15 minutes)

```sql
-- Distance
SELECT ST_Distance(ST_Point(0, 0), ST_Point(3, 4));
-- Result: 5.0

-- Area (requires polygon)
SELECT ST_Area(ST_GeomFromText('POLYGON ((0 0, 4 0, 4 3, 0 3, 0 0))'));
-- Result: 12.0

-- Length (for linestrings)
SELECT ST_Length(ST_GeomFromText('LINESTRING (0 0, 3 4)'));
-- Result: 5.0

-- Perimeter
SELECT ST_Perimeter(ST_GeomFromText('POLYGON ((0 0, 4 0, 4 3, 0 3, 0 0))'));
-- Result: 14.0
```

---

## 2.6 Spatial Transformations (20 minutes)

```sql
-- Buffer
SELECT ST_AsText(ST_Buffer(ST_Point(0, 0), 1));

-- Centroid
SELECT ST_AsText(ST_Centroid(
    ST_GeomFromText('POLYGON ((0 0, 4 0, 4 4, 0 4, 0 0))')
));
-- Result: POINT (2 2)

-- Envelope (bounding box)
SELECT ST_AsText(ST_Envelope(
    ST_GeomFromText('LINESTRING (0 0, 3 5, 1 8)')
));

-- Intersection
SELECT ST_AsText(ST_Intersection(
    ST_GeomFromText('POLYGON ((0 0, 2 0, 2 2, 0 2, 0 0))'),
    ST_GeomFromText('POLYGON ((1 1, 3 1, 3 3, 1 3, 1 1))')
));

-- Union
SELECT ST_AsText(ST_Union(
    ST_GeomFromText('POLYGON ((0 0, 2 0, 2 2, 0 2, 0 0))'),
    ST_GeomFromText('POLYGON ((1 1, 3 1, 3 3, 1 3, 1 1))')
));

-- Simplify
SELECT ST_AsText(ST_Simplify(
    ST_GeomFromText('LINESTRING (0 0, 1 0.1, 2 0, 3 0)'),
    0.2
));
```

---

## 2.7 Output Formats (15 minutes)

```sql
-- Well-Known Text (WKT)
SELECT ST_AsText(geom) FROM geometries;

-- Well-Known Binary (WKB)
SELECT ST_AsBinary(geom) FROM geometries;

-- GeoJSON
SELECT ST_AsGeoJSON(geom) FROM geometries;

-- Export to file
COPY (
    SELECT ST_AsGeoJSON(geom) as geometry, *
    FROM locations
) TO 'output.geojson' (FORMAT JSON);
```

---

## Hands-on Exercise 2: Building a Restaurant Database

```python
import duckdb

conn = duckdb.connect('restaurants.duckdb')
conn.execute("INSTALL spatial; LOAD spatial;")

# Create restaurant table
conn.execute("""
    CREATE TABLE restaurants (
        id INTEGER PRIMARY KEY,
        name VARCHAR,
        cuisine VARCHAR,
        rating FLOAT,
        location GEOMETRY
    )
""")

# Insert restaurants
conn.execute("""
    INSERT INTO restaurants VALUES
        (1, 'Pizza Palace', 'Italian', 4.5, ST_Point(-73.9857, 40.7484)),
        (2, 'Sushi Bar', 'Japanese', 4.8, ST_Point(-73.9851, 40.7489)),
        (3, 'Burger Joint', 'American', 4.2, ST_Point(-73.9862, 40.7479)),
        (4, 'Taco Stand', 'Mexican', 4.6, ST_Point(-73.9845, 40.7492)),
        (5, 'Thai Kitchen', 'Thai', 4.7, ST_Point(-73.9869, 40.7475))
""")

# Query 1: Find restaurants within 500m of a point
result = conn.execute("""
    SELECT
        name,
        cuisine,
        rating,
        ST_Distance(
            location,
            ST_Point(-73.9857, 40.7484)
        ) * 111000 as distance_m
    FROM restaurants
    WHERE ST_DWithin(location, ST_Point(-73.9857, 40.7484), 0.005)
    ORDER BY distance_m
""").df()

print(result)

# Query 2: Find the nearest restaurant to a location
result = conn.execute("""
    SELECT name, cuisine
    FROM restaurants
    ORDER BY ST_Distance(location, ST_Point(-73.986, 40.748))
    LIMIT 1
""").fetchone()

print(f"Nearest restaurant: {result[0]}")

# Query 3: Create service areas (500m buffers)
conn.execute("""
    CREATE TABLE service_areas AS
    SELECT
        id,
        name,
        ST_Buffer(location, 0.005) as service_area
    FROM restaurants
""")
```

---

## Key Takeaways

- SedonaDB supports all OGC geometry types
- Load data from CSV, GeoJSON, Shapefile, GeoParquet
- Spatial predicates test geometric relationships
- Measurements return distance, area, length
- Transformations create new geometries
- CRS transformations essential for accurate calculations
- Multiple output formats for interoperability

---

**Previous:** [Module 1](../module1/README.md) | **Next:** [Module 3](../module3/README.md)
