# Module 2: Spatial Data Fundamentals
**Duration:** 1.5 hours

## Overview
This module covers the fundamentals of geospatial data types, formats, coordinate systems, and how to load and create spatial datasets in Apache Sedona.

---

## 2.1 Geospatial Data Types (20 minutes)

### OGC Simple Features Specification
Sedona implements the Open Geospatial Consortium (OGC) Simple Features specification.

### Basic Geometry Types

**1. Point**
```
POINT (30 10)
```
- Single coordinate pair (x, y) or (longitude, latitude)
- Used for: locations, events, sensors

**2. LineString**
```
LINESTRING (30 10, 10 30, 40 40)
```
- Ordered sequence of points
- Used for: roads, routes, rivers, boundaries

**3. Polygon**
```
POLYGON ((30 10, 40 40, 20 40, 10 20, 30 10))
```
- Closed ring of points (first = last)
- Can have holes (inner rings)
- Used for: buildings, parcels, zones, countries

**4. MultiPoint**
```
MULTIPOINT ((10 40), (40 30), (20 20), (30 10))
```
- Collection of points
- Used for: multiple locations, scattered features

**5. MultiLineString**
```
MULTILINESTRING ((10 10, 20 20, 10 40), (40 40, 30 30, 40 20, 30 10))
```
- Collection of linestrings
- Used for: highway systems, river networks

**6. MultiPolygon**
```
MULTIPOLYGON (((30 20, 45 40, 10 40, 30 20)), ((15 5, 40 10, 10 20, 5 10, 15 5)))
```
- Collection of polygons
- Used for: archipelagos, fragmented regions

**7. GeometryCollection**
```
GEOMETRYCOLLECTION (POINT (40 10), LINESTRING (10 10, 20 20, 10 40))
```
- Mixed geometry types in one object
- Used for: complex features

### Polygon with Holes Example
```
POLYGON ((35 10, 45 45, 15 40, 10 20, 35 10),
         (20 30, 35 35, 30 20, 20 30))
```
- Outer ring: (35 10, 45 45, 15 40, 10 20, 35 10)
- Inner ring (hole): (20 30, 35 35, 30 20, 20 30)

---

## 2.2 Spatial Data Formats (20 minutes)

### Well-Known Text (WKT)
Human-readable text representation.

```python
from sedona.sql import st_functions as ST

# Create geometry from WKT
df = sedona.sql("""
    SELECT ST_GeomFromWKT('POINT (30 10)') as geom
""")
```

### Well-Known Binary (WKB)
Binary representation for efficient storage.

```python
# Convert to WKB
df = sedona.sql("""
    SELECT ST_AsBinary(ST_GeomFromWKT('POINT (30 10)')) as wkb
""")
```

### GeoJSON
JSON format for web mapping.

```json
{
  "type": "Feature",
  "geometry": {
    "type": "Point",
    "coordinates": [30, 10]
  },
  "properties": {
    "name": "Location A"
  }
}
```

### Shapefile
ESRI format (multi-file: .shp, .shx, .dbf, .prj).

```python
from sedona.core.formatMapper.shapefileParser import ShapefileReader

# Read shapefile
spatial_rdd = ShapefileReader.readToGeometryRDD(sc, "/path/to/file.shp")
```

### GeoParquet
Columnar format optimized for analytics.

```python
# Read GeoParquet
df = sedona.read.format("geoparquet").load("/path/to/file.parquet")
```

### CSV with Coordinates
```csv
id,name,longitude,latitude
1,Location A,30,10
2,Location B,25,15
```

```python
# Load CSV and create geometries
df = sedona.read.csv("/path/to/points.csv", header=True)
df = df.selectExpr("id", "name", "ST_Point(CAST(longitude AS Decimal(24,20)), CAST(latitude AS Decimal(24,20))) as geometry")
```

---

## 2.3 Coordinate Reference Systems (20 minutes)

### What is a CRS?
A Coordinate Reference System defines how coordinates map to locations on Earth.

### Common CRS

**WGS84 (EPSG:4326)**
- Geographic coordinate system
- Units: degrees
- Range: Longitude [-180, 180], Latitude [-90, 90]
- Used by GPS, most web maps

**Web Mercator (EPSG:3857)**
- Projected coordinate system
- Units: meters
- Used by Google Maps, OpenStreetMap

**UTM (Universal Transverse Mercator)**
- 60 zones worldwide
- Units: meters
- High accuracy within zone

### CRS in Sedona

```python
# Create geometry with specific SRID
df = sedona.sql("""
    SELECT ST_SetSRID(ST_GeomFromWKT('POINT (30 10)'), 4326) as geom
""")

# Transform between CRS
df = sedona.sql("""
    SELECT ST_Transform(
        ST_SetSRID(ST_GeomFromWKT('POINT (-73.935242 40.730610)'), 4326),
        3857
    ) as geom_3857
""")
```

### Important Considerations
- Always know your data's CRS
- Transform to common CRS before spatial operations
- Distance calculations require projected CRS (meters)
- Avoid mixing CRS in same analysis

---

## 2.4 Loading Spatial Data (25 minutes)

### Method 1: From CSV with Coordinates

```python
# Load CSV
df = sedona.read.csv("data/points.csv", header=True)

# Create Point geometries
from pyspark.sql.functions import expr

df = df.withColumn("geometry",
    expr("ST_Point(CAST(lon AS Decimal(24,20)), CAST(lat AS Decimal(24,20)))"))

df.printSchema()
df.show(5)
```

### Method 2: From WKT Column

```python
# Data with WKT column
df = sedona.read.csv("data/geometries.csv", header=True)

# Parse WKT
df = df.withColumn("geometry", expr("ST_GeomFromWKT(wkt)"))

df.createOrReplaceTempView("spatial_data")
```

### Method 3: From GeoJSON

```python
# Read GeoJSON
from sedona.core.formatMapper import GeoJsonReader

spatial_rdd = GeoJsonReader.readToGeometryRDD(sc, "data/features.geojson")

# Convert to DataFrame
df = Adapter.toDf(spatial_rdd, sedona)
df.show()
```

### Method 4: From Shapefile

```python
from sedona.core.formatMapper.shapefileParser import ShapefileReader
from sedona.utils.adapter import Adapter

# Read shapefile
spatial_rdd = ShapefileReader.readToGeometryRDD(sc, "data/boundaries.shp")

# Convert to DataFrame
df = Adapter.toDf(spatial_rdd, sedona)
df.show()
```

### Method 5: From GeoParquet

```python
# Most efficient for big data
df = sedona.read.format("geoparquet") \
    .load("data/spatial_data.parquet")

df.printSchema()
```

### Method 6: From PostGIS

```python
# JDBC connection
jdbc_url = "jdbc:postgresql://localhost:5432/geodb"
properties = {
    "user": "user",
    "password": "pass",
    "driver": "org.postgresql.Driver"
}

# Read spatial table
df = sedona.read.jdbc(
    url=jdbc_url,
    table="(SELECT id, name, ST_AsText(geom) as wkt FROM locations) as subq",
    properties=properties
)

# Parse WKT
df = df.withColumn("geometry", expr("ST_GeomFromWKT(wkt)"))
```

---

## 2.5 Creating Spatial DataFrames (15 minutes)

### From Scratch

```python
from pyspark.sql import Row

# Create data
data = [
    Row(id=1, name="A", wkt="POINT (30 10)"),
    Row(id=2, name="B", wkt="LINESTRING (30 10, 10 30, 40 40)"),
    Row(id=3, name="C", wkt="POLYGON ((30 10, 40 40, 20 40, 10 20, 30 10))")
]

# Create DataFrame
df = sedona.createDataFrame(data)

# Add geometry column
df = df.withColumn("geometry", expr("ST_GeomFromWKT(wkt)"))
```

### Programmatic Geometry Creation

```python
# Using ST functions
df = sedona.sql("""
    SELECT
        ST_Point(1.0, 2.0) as point,
        ST_MakeLine(
            ST_Point(0.0, 0.0),
            ST_Point(1.0, 1.0)
        ) as line,
        ST_Buffer(ST_Point(0.0, 0.0), 5.0) as circle
""")
```

### From RDD

```python
from sedona.core.SpatialRDD import PointRDD
from sedona.utils.adapter import Adapter

# Create PointRDD
point_rdd = PointRDD(
    sc,
    "data/points.csv",
    0,  # offset
    FileDataSplitter.CSV,
    True,  # carryInputData
    1  # numPartitions
)

# Convert to DataFrame
df = Adapter.toDf(point_rdd, sedona)
```

---

## 2.6 Spatial Data Serialization (10 minutes)

### Writing Spatial Data

**GeoParquet (Recommended)**
```python
df.write.format("geoparquet") \
    .mode("overwrite") \
    .save("output/spatial_data.parquet")
```

**CSV with WKT**
```python
# Convert geometry to WKT
df_export = df.withColumn("wkt", expr("ST_AsText(geometry)")) \
    .drop("geometry")

df_export.write.csv("output/export.csv", header=True)
```

**GeoJSON**
```python
# Export to GeoJSON
df.selectExpr("ST_AsGeoJSON(geometry) as geojson") \
    .write.text("output/features.geojson")
```

**Shapefile**
```python
from sedona.utils.adapter import Adapter

# Convert to RDD
spatial_rdd = Adapter.toSpatialRdd(df, "geometry")

# Save as shapefile
spatial_rdd.saveAsWKB("output/shapefile")
```

---

## Hands-on Exercise 2: Loading and Transforming Spatial Data

### Objective
Load various spatial data formats, transform CRS, and create geometries.

### Dataset
NYC taxi pickup locations (sample).

### Tasks

**Task 1: Load CSV with Coordinates**
```python
# Load taxi pickup data
df_taxi = sedona.read.csv("data/nyc_taxi_sample.csv", header=True)

# Create Point geometries
df_taxi = df_taxi.withColumn("pickup_point",
    expr("ST_Point(CAST(pickup_longitude AS Decimal(24,20)), CAST(pickup_latitude AS Decimal(24,20)))"))

df_taxi.select("pickup_point").show(5, truncate=False)
```

**Task 2: Transform CRS**
```python
# Set SRID to WGS84 and transform to Web Mercator
df_taxi = df_taxi.withColumn("pickup_point_3857",
    expr("ST_Transform(ST_SetSRID(pickup_point, 4326), 3857)"))

df_taxi.select(
    expr("ST_AsText(pickup_point) as wgs84"),
    expr("ST_AsText(pickup_point_3857) as web_mercator")
).show(5, truncate=False)
```

**Task 3: Load NYC Borough Boundaries**
```python
# Load borough polygons from GeoJSON
from sedona.core.formatMapper import GeoJsonReader

borough_rdd = GeoJsonReader.readToGeometryRDD(sc, "data/nyc_boroughs.geojson")
df_boroughs = Adapter.toDf(borough_rdd, sedona)

df_boroughs.printSchema()
df_boroughs.show()
```

**Task 4: Save as GeoParquet**
```python
# Save transformed data
df_taxi.select("pickup_point_3857") \
    .write.format("geoparquet") \
    .mode("overwrite") \
    .save("output/taxi_pickups.parquet")
```

---

## Code Examples

### Complete Example: Multi-Format Data Pipeline

```python
from sedona.spark import *
from pyspark.sql.functions import expr

# Initialize Sedona
config = SedonaContext.builder().appName('module2').getOrCreate()
sedona = SedonaContext.create(config)

# 1. Load points from CSV
df_points = sedona.read.csv("points.csv", header=True)
df_points = df_points.withColumn("geom",
    expr("ST_Point(CAST(lon AS Decimal(24,20)), CAST(lat AS Decimal(24,20)))"))

# 2. Load polygons from WKT
df_polygons = sedona.read.csv("polygons.csv", header=True)
df_polygons = df_polygons.withColumn("geom", expr("ST_GeomFromWKT(wkt)"))

# 3. Transform CRS
df_points = df_points.withColumn("geom",
    expr("ST_Transform(ST_SetSRID(geom, 4326), 3857)"))
df_polygons = df_polygons.withColumn("geom",
    expr("ST_Transform(ST_SetSRID(geom, 4326), 3857)"))

# 4. Register as temp views
df_points.createOrReplaceTempView("points")
df_polygons.createOrReplaceTempView("polygons")

# 5. Spatial query
result = sedona.sql("""
    SELECT p.id, p.name, poly.region
    FROM points p, polygons poly
    WHERE ST_Contains(poly.geom, p.geom)
""")

# 6. Save results
result.write.format("geoparquet").save("output/results.parquet")
```

---

## Key Takeaways

- Sedona supports all OGC Simple Features geometry types
- Multiple input formats: WKT, WKB, GeoJSON, Shapefile, GeoParquet, CSV
- Always specify and transform CRS appropriately
- GeoParquet is the most efficient format for big data
- ST_GeomFromWKT and ST_Point are primary functions for creating geometries
- Spatial DataFrames integrate seamlessly with Spark SQL

---

## Common Pitfalls

1. **Mixing CRS**: Always transform to common CRS before operations
2. **Wrong column types**: Cast coordinates to Decimal, not String
3. **Invalid geometries**: Validate with ST_IsValid
4. **Missing SRID**: Always set with ST_SetSRID
5. **Large WKT strings**: Use WKB or native formats for efficiency

---

## Quiz Questions

1. What is the difference between WKT and WKB?
2. Which EPSG code represents WGS84?
3. How do you create a Point geometry from longitude and latitude columns?
4. What is the recommended format for storing large spatial datasets in Spark?
5. Name three methods to load spatial data into Sedona.
6. Why is CRS transformation necessary?

---

**Previous:** [Module 1: Introduction](../module1/README.md) | **Next:** [Module 3: Spatial RDDs and Operations](../module3/README.md)
