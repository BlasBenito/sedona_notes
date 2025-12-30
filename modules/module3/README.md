# Module 3: Spatial RDDs and Operations
**Duration:** 1.5 hours

## Overview
This module explores Spatial RDDs, the foundation of Sedona's distributed computing model. You'll learn to create, manipulate, and transform spatial data using RDD-based operations.

---

## 3.1 Introduction to Spatial RDDs (15 minutes)

### What are Spatial RDDs?
Spatial RDDs extend Spark's RDD abstraction to handle geometric data with spatial awareness.

### Key Spatial RDD Types

**PointRDD**
- Collection of Point geometries
- Most common for location data

**PolygonRDD**
- Collection of Polygon geometries
- Used for regions, boundaries, zones

**LineStringRDD**
- Collection of LineString geometries
- Used for roads, routes, paths

**RectangleRDD**
- Collection of Rectangle geometries
- Optimized for bounding boxes

### Architecture

```
┌──────────────────────────────────────┐
│         Spatial RDD                   │
├──────────────────────────────────────┤
│  Partition 1  │ Partition 2 │ Part 3 │
│  [Geom, ...]  │ [Geom, ...] │ [...]  │
├──────────────────────────────────────┤
│  Spatial Index (Optional)             │
│  - R-Tree / Quad-Tree                 │
├──────────────────────────────────────┤
│  Spatial Partitioning (Optional)      │
│  - Grid / KDB-Tree                    │
└──────────────────────────────────────┘
```

### Advantages
- Distributed processing across cluster
- Spatial indexing for query optimization
- Lazy evaluation (like regular RDDs)
- Support for spatial partitioning

---

## 3.2 Creating Spatial RDDs (20 minutes)

### Method 1: From CSV File

```python
from sedona.core.SpatialRDD import PointRDD
from sedona.core.enums import FileDataSplitter

# Create PointRDD from CSV
point_rdd = PointRDD(
    sc,                          # Spark Context
    "data/points.csv",           # Input path
    0,                           # Offset (which column has coordinates)
    FileDataSplitter.CSV,        # File format
    True,                        # Carry other attributes
    2                            # Number of partitions
)

print(f"Total count: {point_rdd.countWithoutDuplicates()}")
```

**CSV Format:**
```csv
-73.935242,40.730610,Location A,2023-01-01
-73.989667,40.751311,Location B,2023-01-02
```

### Method 2: From WKT File

```python
from sedona.core.SpatialRDD import PolygonRDD
from sedona.core.enums import FileDataSplitter

# Create PolygonRDD from WKT
polygon_rdd = PolygonRDD(
    sc,
    "data/polygons.wkt",
    0,                           # WKT column index
    FileDataSplitter.WKT,
    True,
    4
)
```

### Method 3: From Shapefile

```python
from sedona.core.formatMapper.shapefileParser import ShapefileReader

# Read shapefile to SpatialRDD
spatial_rdd = ShapefileReader.readToGeometryRDD(
    sc,
    "data/boundaries.shp"
)

print(f"Geometry count: {spatial_rdd.countWithoutDuplicates()}")
```

### Method 4: From GeoJSON

```python
from sedona.core.formatMapper import GeoJsonReader

# Read GeoJSON
spatial_rdd = GeoJsonReader.readToGeometryRDD(
    sc,
    "data/features.geojson"
)
```

### Method 5: From DataFrame

```python
from sedona.utils.adapter import Adapter

# Convert DataFrame to SpatialRDD
spatial_rdd = Adapter.toSpatialRdd(df, "geometry")

# Convert back to DataFrame
df_new = Adapter.toDf(spatial_rdd, sedona)
```

### Method 6: Programmatically

```python
from shapely.geometry import Point
from sedona.core.SpatialRDD import PointRDD

# Create geometries
points = [Point(i, i*2) for i in range(100)]
point_rdd = PointRDD(sc.parallelize(points))
```

---

## 3.3 Spatial Predicates (25 minutes)

### Overview
Spatial predicates test relationships between geometries.

### DE-9IM Model
Dimensionally Extended 9-Intersection Model defines spatial relationships.

### Common Predicates

**1. ST_Contains**
```python
# Check if polygon contains points
result = sedona.sql("""
    SELECT p.id, poly.name
    FROM points p, polygons poly
    WHERE ST_Contains(poly.geometry, p.geometry)
""")
```

**2. ST_Intersects**
```python
# Find geometries that intersect
result = sedona.sql("""
    SELECT a.id, b.id
    FROM dataset_a a, dataset_b b
    WHERE ST_Intersects(a.geometry, b.geometry)
""")
```

**3. ST_Within**
```python
# Points within distance of a location
result = sedona.sql("""
    SELECT id, name
    FROM locations
    WHERE ST_Within(
        geometry,
        ST_Buffer(ST_Point(-73.935242, 40.730610), 0.01)
    )
""")
```

**4. ST_Overlaps**
```python
# Polygons that overlap (not complete containment)
result = sedona.sql("""
    SELECT a.id, b.id
    FROM parcels a, zones b
    WHERE ST_Overlaps(a.geometry, b.geometry)
""")
```

**5. ST_Touches**
```python
# Geometries that touch at boundary
result = sedona.sql("""
    SELECT a.id, b.id
    FROM parcels a, parcels b
    WHERE ST_Touches(a.geometry, b.geometry) AND a.id < b.id
""")
```

**6. ST_Crosses**
```python
# Lines that cross polygons
result = sedona.sql("""
    SELECT road.id, zone.name
    FROM roads road, zones zone
    WHERE ST_Crosses(road.geometry, zone.geometry)
""")
```

**7. ST_Disjoint**
```python
# Geometries with no intersection
result = sedona.sql("""
    SELECT a.id, b.id
    FROM dataset_a a, dataset_b b
    WHERE ST_Disjoint(a.geometry, b.geometry)
""")
```

### Predicate Relationships

```
         A          B
      ┌─────┐   ┌─────┐
      │  A  │   │  B  │
      └─────┘   └─────┘

    Disjoint: No overlap

         A
      ┌─────┐
      │  ┌──┼──┐
      └──┼──┘  │
         └─────┘
           B

    Intersects: Any overlap

         A
      ┌─────────┐
      │  ┌───┐  │
      │  │ B │  │
      │  └───┘  │
      └─────────┘

    Contains: A completely contains B
```

---

## 3.4 Spatial Transformations (25 minutes)

### 1. ST_Buffer
Creates a polygon representing all points within a distance.

```python
# Buffer points by 1000 meters (assuming meters CRS)
df = sedona.sql("""
    SELECT id, ST_Buffer(geometry, 1000.0) as buffer_geom
    FROM locations
""")
```

**Use cases:**
- Proximity zones
- Service areas
- Impact analysis

### 2. ST_Envelope / ST_Bounds
Minimum bounding rectangle.

```python
# Get bounding box
df = sedona.sql("""
    SELECT id, ST_Envelope(geometry) as bbox
    FROM complex_geometries
""")
```

### 3. ST_ConvexHull
Smallest convex polygon containing all points.

```python
# Convex hull
df = sedona.sql("""
    SELECT id, ST_ConvexHull(geometry) as hull
    FROM point_clusters
""")
```

### 4. ST_Centroid
Geometric center of a geometry.

```python
# Find centroids
df = sedona.sql("""
    SELECT id, ST_Centroid(geometry) as center
    FROM polygons
""")
```

### 5. ST_Union
Merge multiple geometries into one.

```python
# Union all geometries in group
df = sedona.sql("""
    SELECT region, ST_Union_Aggr(geometry) as merged_geom
    FROM parcels
    GROUP BY region
""")
```

### 6. ST_Intersection
Geometric intersection of two geometries.

```python
# Find intersection area
df = sedona.sql("""
    SELECT
        a.id,
        b.id,
        ST_Intersection(a.geometry, b.geometry) as overlap
    FROM zones a, zones b
    WHERE a.id != b.id AND ST_Intersects(a.geometry, b.geometry)
""")
```

### 7. ST_Difference
Geometry A minus geometry B.

```python
# Remove overlap
df = sedona.sql("""
    SELECT ST_Difference(a.geometry, b.geometry) as remaining
    FROM parcel a, exclusion b
    WHERE ST_Intersects(a.geometry, b.geometry)
""")
```

### 8. ST_SimplifyPreserveTopology
Reduce geometry complexity.

```python
# Simplify for visualization
df = sedona.sql("""
    SELECT id, ST_SimplifyPreserveTopology(geometry, 0.01) as simple_geom
    FROM detailed_boundaries
""")
```

---

## 3.5 Coordinate Reference System Transformations (15 minutes)

### Setting SRID

```python
# Set spatial reference ID
df = sedona.sql("""
    SELECT ST_SetSRID(geometry, 4326) as geom_with_srid
    FROM locations
""")
```

### Transforming CRS

```python
# Transform from WGS84 (4326) to Web Mercator (3857)
df = sedona.sql("""
    SELECT
        id,
        geometry as original,
        ST_Transform(ST_SetSRID(geometry, 4326), 3857) as transformed
    FROM locations
""")
```

### Practical Example: Distance Calculation

```python
# WRONG: Distance in degrees (meaningless)
df = sedona.sql("""
    SELECT ST_Distance(
        ST_Point(-73.935242, 40.730610),
        ST_Point(-73.989667, 40.751311)
    ) as distance_degrees
""")

# CORRECT: Distance in meters
df = sedona.sql("""
    SELECT ST_Distance(
        ST_Transform(ST_SetSRID(ST_Point(-73.935242, 40.730610), 4326), 3857),
        ST_Transform(ST_SetSRID(ST_Point(-73.989667, 40.751311), 4326), 3857)
    ) as distance_meters
""")
```

### Using Geography for Spherical Distance

```python
# Haversine distance (approximate spherical distance)
df = sedona.sql("""
    SELECT ST_DistanceSphere(
        ST_Point(-73.935242, 40.730610),
        ST_Point(-73.989667, 40.751311)
    ) as distance_meters
""")
```

---

## 3.6 RDD Operations and Caching (10 minutes)

### Basic RDD Operations

```python
from sedona.core.SpatialRDD import PointRDD

# Create RDD
point_rdd = PointRDD(sc, "data/points.csv", 0, FileDataSplitter.CSV, True)

# Count
total = point_rdd.countWithoutDuplicates()

# Approximate count (faster)
approx = point_rdd.approximateTotalCount

# Boundary
envelope = point_rdd.boundaryEnvelope
print(f"Bounds: {envelope}")

# Sample
sample = point_rdd.rawSpatialRDD.take(10)
```

### Filtering

```python
# Filter geometries
from shapely.geometry import Point

def filter_func(geom):
    # Keep only points in specific region
    return -74.0 <= geom.x <= -73.9 and 40.7 <= geom.y <= 40.8

filtered_rdd = point_rdd.rawSpatialRDD.filter(filter_func)
```

### Caching and Persistence

```python
# Cache in memory (recommended for iterative operations)
point_rdd.rawSpatialRDD.cache()

# Persist to disk
from pyspark import StorageLevel
point_rdd.rawSpatialRDD.persist(StorageLevel.MEMORY_AND_DISK)

# Unpersist when done
point_rdd.rawSpatialRDD.unpersist()
```

### Converting Between RDD and DataFrame

```python
from sedona.utils.adapter import Adapter

# RDD -> DataFrame
df = Adapter.toDf(spatial_rdd, sedona)

# DataFrame -> RDD
spatial_rdd = Adapter.toSpatialRdd(df, "geometry")
```

---

## Hands-on Exercise 3: Spatial Operations Pipeline

### Objective
Build a pipeline to analyze point-in-polygon relationships and create buffers.

### Scenario
You have customer locations and delivery zones. Find:
1. Customers in each zone
2. Create 5km buffers around each customer
3. Find zones that overlap with customer buffers

### Code Solution

```python
from sedona.spark import *
from pyspark.sql.functions import expr

# Initialize
config = SedonaContext.builder().appName('module3').getOrCreate()
sedona = SedonaContext.create(config)

# 1. Load customer locations
customers = sedona.read.csv("data/customers.csv", header=True)
customers = customers.withColumn("location",
    expr("ST_Point(CAST(lon AS Decimal(24,20)), CAST(lat AS Decimal(24,20)))"))
customers = customers.withColumn("location",
    expr("ST_Transform(ST_SetSRID(location, 4326), 3857)"))

# 2. Load delivery zones
zones = sedona.read.option("multiLine", "true").json("data/zones.geojson")
zones = zones.withColumn("zone_geom", expr("ST_GeomFromGeoJSON(geometry)"))
zones = zones.withColumn("zone_geom",
    expr("ST_Transform(ST_SetSRID(zone_geom, 4326), 3857)"))

# 3. Register temp views
customers.createOrReplaceTempView("customers")
zones.createOrReplaceTempView("zones")

# 4. Find customers in each zone
result1 = sedona.sql("""
    SELECT
        z.zone_id,
        z.zone_name,
        COUNT(c.customer_id) as customer_count
    FROM zones z
    LEFT JOIN customers c ON ST_Contains(z.zone_geom, c.location)
    GROUP BY z.zone_id, z.zone_name
    ORDER BY customer_count DESC
""")

print("Customers per zone:")
result1.show()

# 5. Create 5km buffers around customers
customers_buffered = sedona.sql("""
    SELECT
        customer_id,
        customer_name,
        location,
        ST_Buffer(location, 5000.0) as buffer_5km
    FROM customers
""")

customers_buffered.createOrReplaceTempView("customers_buffered")

# 6. Find zones overlapping with customer buffers
result2 = sedona.sql("""
    SELECT DISTINCT
        c.customer_id,
        c.customer_name,
        z.zone_id,
        z.zone_name,
        ST_Area(ST_Intersection(c.buffer_5km, z.zone_geom)) as overlap_area
    FROM customers_buffered c, zones z
    WHERE ST_Intersects(c.buffer_5km, z.zone_geom)
    ORDER BY c.customer_id, overlap_area DESC
""")

print("\nZone overlaps with 5km customer buffers:")
result2.show()

# 7. Export results
result2.write.format("geoparquet") \
    .mode("overwrite") \
    .save("output/customer_zone_overlaps.parquet")
```

---

## Performance Tips

1. **Use appropriate CRS**: Transform to projected CRS for distance operations
2. **Cache strategically**: Cache RDDs used multiple times
3. **Filter early**: Apply filters before expensive operations
4. **Simplify geometries**: Use ST_SimplifyPreserveTopology for large datasets
5. **Partition wisely**: Balance partition size (aim for 128MB-256MB)

---

## Key Takeaways

- Spatial RDDs are the foundation of distributed geospatial processing
- Three main types: PointRDD, PolygonRDD, LineStringRDD
- Spatial predicates test geometric relationships
- Transformations create new geometries (buffer, union, etc.)
- Always transform to appropriate CRS for accurate measurements
- Cache RDDs for iterative operations
- Convert between RDD and DataFrame as needed

---

## Common Errors and Solutions

**Error:** "Geometry has no SRID"
```python
# Solution: Always set SRID
df = df.withColumn("geom", expr("ST_SetSRID(geom, 4326)"))
```

**Error:** "Distance in degrees is meaningless"
```python
# Solution: Transform to projected CRS or use ST_DistanceSphere
df = df.withColumn("geom", expr("ST_Transform(ST_SetSRID(geom, 4326), 3857)"))
```

**Error:** "OutOfMemoryError"
```python
# Solution: Increase partitions
spatial_rdd.spatialPartitioning(GridType.KDBTREE, 100)
```

---

## Quiz Questions

1. What are the three main types of Spatial RDDs?
2. What's the difference between ST_Contains and ST_Within?
3. Why must you transform to a projected CRS for distance calculations?
4. What does ST_Buffer create?
5. How do you cache a Spatial RDD?
6. What's the difference between ST_Intersects and ST_Overlaps?

---

**Previous:** [Module 2: Spatial Data Fundamentals](../module2/README.md) | **Next:** [Module 4: Spatial Queries and Joins](../module4/README.md)
