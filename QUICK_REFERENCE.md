# Apache Sedona Quick Reference Guide

## Initialization

```python
from sedona.spark import *

config = SedonaContext.builder() \
    .appName('my-app') \
    .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
    .config("spark.kryo.registrator", "org.apache.sedona.core.serde.SedonaKryoRegistrator") \
    .getOrCreate()

sedona = SedonaContext.create(config)
```

---

## Geometry Constructors

```sql
-- Point
ST_Point(longitude, latitude)
ST_PointFromText('POINT (30 10)', 4326)

-- LineString
ST_LineFromText('LINESTRING (30 10, 10 30, 40 40)', 4326)
ST_MakeLine(point1, point2)

-- Polygon
ST_PolygonFromText('POLYGON ((30 10, 40 40, 20 40, 10 20, 30 10))', 4326)
ST_MakePolygon(linestring)

-- From WKT/WKB
ST_GeomFromWKT(wkt_string)
ST_GeomFromWKB(wkb_binary)

-- From GeoJSON
ST_GeomFromGeoJSON(geojson_string)

-- Envelope/BBox
ST_MakeEnvelope(minX, minY, maxX, maxY)
ST_PolygonFromEnvelope(minX, minY, maxX, maxY)
```

---

## Spatial Predicates (Return Boolean)

```sql
ST_Contains(geom1, geom2)       -- geom1 completely contains geom2
ST_Within(geom1, geom2)         -- geom1 completely within geom2
ST_Intersects(geom1, geom2)     -- geometries share any space
ST_Crosses(geom1, geom2)        -- geometries cross each other
ST_Overlaps(geom1, geom2)       -- geometries overlap (not complete containment)
ST_Touches(geom1, geom2)        -- geometries touch at boundary only
ST_Disjoint(geom1, geom2)       -- geometries don't intersect
ST_Equals(geom1, geom2)         -- geometries are spatially equal
ST_DWithin(geom1, geom2, dist)  -- geometries within distance
```

---

## Spatial Measurements

```sql
ST_Distance(geom1, geom2)       -- Distance between geometries
ST_Area(geometry)               -- Area of polygon
ST_Length(geometry)             -- Length of linestring
ST_Perimeter(geometry)          -- Perimeter of polygon
ST_DistanceSphere(geom1, geom2) -- Spherical distance (approximate)
```

---

## Spatial Transformations

```sql
-- Buffer
ST_Buffer(geometry, distance)

-- Centroid
ST_Centroid(geometry)

-- Envelope/Bounds
ST_Envelope(geometry)
ST_Bounds(geometry)

-- Convex Hull
ST_ConvexHull(geometry)

-- Union
ST_Union(geom1, geom2)
ST_Union_Aggr(geometry)         -- Aggregate function

-- Intersection
ST_Intersection(geom1, geom2)

-- Difference
ST_Difference(geom1, geom2)

-- Symmetric Difference
ST_SymDifference(geom1, geom2)

-- Simplify
ST_Simplify(geometry, tolerance)
ST_SimplifyPreserveTopology(geometry, tolerance)
```

---

## Coordinate Reference Systems

```sql
-- Set SRID
ST_SetSRID(geometry, srid)

-- Get SRID
ST_SRID(geometry)

-- Transform CRS
ST_Transform(geometry, target_srid)

-- Common SRID values:
-- 4326 = WGS84 (GPS, lat/lon in degrees)
-- 3857 = Web Mercator (Google Maps, OpenStreetMap)
-- 32633 = UTM Zone 33N (meters, Europe)
```

---

## Accessors

```sql
-- Coordinates
ST_X(point)                     -- X coordinate
ST_Y(point)                     -- Y coordinate

-- Geometry components
ST_GeometryN(geometry, n)       -- Nth geometry (0-indexed)
ST_NumGeometries(geometry)      -- Number of geometries
ST_PointN(linestring, n)        -- Nth point (0-indexed)
ST_NumPoints(geometry)          -- Number of points
ST_ExteriorRing(polygon)        -- Outer ring
ST_InteriorRingN(polygon, n)    -- Nth inner ring

-- Geometry type
ST_GeometryType(geometry)       -- Type as string
ST_Dimension(geometry)          -- Dimension (0=point, 1=line, 2=polygon)

-- Validation
ST_IsValid(geometry)            -- Check if valid
ST_IsSimple(geometry)           -- Check if simple
ST_IsEmpty(geometry)            -- Check if empty
```

---

## Output Formats

```sql
ST_AsText(geometry)             -- WKT
ST_AsGeoJSON(geometry)          -- GeoJSON
ST_AsBinary(geometry)           -- WKB
ST_AsEWKT(geometry)             -- Extended WKT (with SRID)
```

---

## Loading Data

### CSV with Coordinates

```python
df = sedona.read.csv("points.csv", header=True)
df = df.withColumn("geometry",
    expr("ST_Point(CAST(lon AS Decimal(24,20)), CAST(lat AS Decimal(24,20)))"))
```

### WKT Column

```python
df = sedona.read.csv("data.csv", header=True)
df = df.withColumn("geometry", expr("ST_GeomFromWKT(wkt)"))
```

### GeoJSON

```python
from sedona.core.formatMapper import GeoJsonReader
from sedona.utils.adapter import Adapter

spatial_rdd = GeoJsonReader.readToGeometryRDD(sc, "data.geojson")
df = Adapter.toDf(spatial_rdd, sedona)
```

### Shapefile

```python
from sedona.core.formatMapper.shapefileParser import ShapefileReader
from sedona.utils.adapter import Adapter

spatial_rdd = ShapefileReader.readToGeometryRDD(sc, "data.shp")
df = Adapter.toDf(spatial_rdd, sedona)
```

### GeoParquet

```python
df = sedona.read.format("geoparquet").load("data.parquet")
```

---

## Spatial RDD Operations

```python
from sedona.core.SpatialRDD import PointRDD, PolygonRDD
from sedona.core.enums import IndexType, GridType, FileDataSplitter

# Create PointRDD
point_rdd = PointRDD(sc, "points.csv", 0, FileDataSplitter.CSV, True)

# Spatial partitioning
point_rdd.spatialPartitioning(GridType.KDBTREE, 200)

# Build index
point_rdd.buildIndex(IndexType.RTREE, True)

# Convert to DataFrame
from sedona.utils.adapter import Adapter
df = Adapter.toDf(point_rdd, sedona)

# Convert DataFrame to RDD
spatial_rdd = Adapter.toSpatialRdd(df, "geometry")
```

---

## Spatial Joins

### SQL Join

```python
result = sedona.sql("""
    SELECT a.id, b.name
    FROM points a
    JOIN polygons b ON ST_Contains(b.geometry, a.geometry)
""")
```

### RDD Join

```python
from sedona.core.spatialOperator import JoinQuery

# Partition both RDDs
point_rdd.spatialPartitioning(GridType.KDBTREE, 200)
polygon_rdd.spatialPartitioning(point_rdd.getPartitioner())

# Build index
point_rdd.buildIndex(IndexType.RTREE, True)

# Execute join
result = JoinQuery.SpatialJoinQuery(
    point_rdd,
    polygon_rdd,
    True,   # useIndex
    False   # considerBoundaryIntersection
)
```

---

## Range Queries

```python
from sedona.core.spatialOperator import RangeQuery
from sedona.core.geom.envelope import Envelope

# Define query window
query_envelope = Envelope(-74.1, -73.9, 40.6, 40.8)

# Execute range query
result = RangeQuery.SpatialRangeQuery(
    spatial_rdd,
    query_envelope,
    False,  # considerBoundaryIntersection
    False   # usingIndex
)
```

---

## KNN Queries

```python
from sedona.core.spatialOperator import KNNQuery
from shapely.geometry import Point

query_point = Point(-73.935242, 40.730610)
k = 10

result = KNNQuery.SpatialKnnQuery(spatial_rdd, query_point, k, False)
```

---

## Common Patterns

### Point in Polygon

```sql
SELECT p.id, poly.name
FROM points p
JOIN polygons poly ON ST_Contains(poly.geometry, p.geometry)
```

### Distance Query

```sql
SELECT id, name,
    ST_Distance(
        ST_Transform(ST_SetSRID(geometry, 4326), 3857),
        ST_Transform(ST_SetSRID(ST_Point(-73.935242, 40.730610), 4326), 3857)
    ) as distance_m
FROM locations
WHERE ST_DWithin(
    ST_Transform(ST_SetSRID(geometry, 4326), 3857),
    ST_Transform(ST_SetSRID(ST_Point(-73.935242, 40.730610), 4326), 3857),
    1000.0
)
ORDER BY distance_m
```

### Buffer Analysis

```sql
SELECT
    id,
    ST_Buffer(
        ST_Transform(ST_SetSRID(geometry, 4326), 3857),
        500.0
    ) as buffer_500m
FROM locations
```

### Grid-based Aggregation

```sql
SELECT
    FLOOR(ST_X(geometry) / 0.01) * 0.01 as grid_x,
    FLOOR(ST_Y(geometry) / 0.01) * 0.01 as grid_y,
    COUNT(*) as count
FROM points
GROUP BY grid_x, grid_y
```

---

## Performance Tips

```python
# 1. Transform to projected CRS for distance calculations
df = df.withColumn("geom_3857",
    expr("ST_Transform(ST_SetSRID(geometry, 4326), 3857)"))

# 2. Cache frequently used data
df.cache()

# 3. Broadcast small tables
from pyspark.sql.functions import broadcast
result = large_df.join(broadcast(small_df), condition)

# 4. Filter early
result = df.filter("category = 'A'").join(other_df, condition)

# 5. Simplify complex geometries
df = df.withColumn("simple_geom",
    expr("ST_SimplifyPreserveTopology(geometry, 0.001)"))

# 6. Use appropriate partitions
df = df.repartition(200)

# 7. Validate geometries
df = df.filter(expr("ST_IsValid(geometry)"))
```

---

## Common SRID Values

| SRID | Name | Type | Units | Coverage |
|------|------|------|-------|----------|
| 4326 | WGS84 | Geographic | Degrees | Global |
| 3857 | Web Mercator | Projected | Meters | Global |
| 2163 | US National Atlas | Projected | Meters | USA |
| 32633 | UTM Zone 33N | Projected | Meters | Europe |
| 32618 | UTM Zone 18N | Projected | Meters | Eastern USA |

---

## Troubleshooting

### Problem: Wrong distance values
```python
# Wrong (degrees)
ST_Distance(geom1, geom2)

# Correct (meters)
ST_Distance(
    ST_Transform(ST_SetSRID(geom1, 4326), 3857),
    ST_Transform(ST_SetSRID(geom2, 4326), 3857)
)
```

### Problem: Slow spatial joins
```python
# Add spatial partitioning and indexing
rdd1.spatialPartitioning(GridType.KDBTREE, 200)
rdd2.spatialPartitioning(rdd1.getPartitioner())
rdd1.buildIndex(IndexType.RTREE, True)
```

### Problem: Invalid geometries
```python
# Check and fix
df = df.filter(expr("ST_IsValid(geometry)"))
df = df.withColumn("geometry", expr("ST_MakeValid(geometry)"))
```

### Problem: Out of memory
```python
# Increase partitions
df = df.repartition(400)

# Or increase executor memory
spark.conf.set("spark.executor.memory", "16g")
```

---

## Quick Start Template

```python
from sedona.spark import *
from pyspark.sql.functions import expr

# Initialize
config = SedonaContext.builder().appName('app').getOrCreate()
sedona = SedonaContext.create(config)

# Load data
df = sedona.read.csv("data.csv", header=True)
df = df.withColumn("geometry",
    expr("ST_Point(CAST(lon AS Decimal(24,20)), CAST(lat AS Decimal(24,20)))"))

# Transform CRS
df = df.withColumn("geometry",
    expr("ST_Transform(ST_SetSRID(geometry, 4326), 3857)"))

# Register view
df.createOrReplaceTempView("locations")

# Query
result = sedona.sql("""
    SELECT * FROM locations
    WHERE ST_DWithin(geometry, ST_Point(0, 0), 1000)
""")

# Export
result.write.format("geoparquet").save("output.parquet")
```

---

**For detailed explanations, see the full course modules.**
