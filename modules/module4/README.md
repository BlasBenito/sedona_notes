# Module 4: Spatial Queries and Joins
**Duration:** 2 hours

## Overview
This module covers the most important operations in distributed geospatial processing: spatial queries and joins. You'll learn to efficiently query and combine large spatial datasets.

---

## 4.1 Spatial Query Fundamentals (15 minutes)

### Types of Spatial Queries

**1. Range Queries**
- Find all geometries in a spatial range
- Example: Points within a polygon, features in bounding box

**2. K-Nearest Neighbors (KNN)**
- Find K closest geometries to a reference point
- Example: 5 nearest restaurants to current location

**3. Distance Queries**
- Find all geometries within distance threshold
- Example: All stores within 5km

**4. Spatial Joins**
- Combine datasets based on spatial relationships
- Example: Join customer points with delivery zones

### Query Complexity

```
┌────────────────────────────────────────┐
│  Query Type          │  Complexity     │
├────────────────────────────────────────┤
│  Bounding Box        │  O(n)           │
│  Point-in-Polygon    │  O(n*m)         │
│  Spatial Join        │  O(n*m)         │
│  KNN (naive)         │  O(n*log k)     │
│  With Index          │  Much faster!   │
└────────────────────────────────────────┘
```

---

## 4.2 Range Queries (25 minutes)

### Bounding Box Query

```python
from sedona.core.geom.envelope import Envelope

# Define search area
query_envelope = Envelope(-74.1, -73.9, 40.6, 40.8)

# Query using DataFrame
result = sedona.sql("""
    SELECT id, name, ST_AsText(geometry) as geom
    FROM locations
    WHERE ST_Intersects(
        geometry,
        ST_PolygonFromEnvelope(-74.1, 40.6, -73.9, 40.8)
    )
""")

result.show()
```

### Point-in-Polygon Query

```python
# Find all points inside a polygon
result = sedona.sql("""
    SELECT p.id, p.name
    FROM points p
    WHERE ST_Contains(
        ST_GeomFromWKT('POLYGON ((-74 40, -74 41, -73 41, -73 40, -74 40))'),
        p.geometry
    )
""")
```

### Using RDD API

```python
from sedona.core.SpatialRDD import PointRDD, PolygonRDD
from sedona.core.spatialOperator import RangeQuery
from sedona.core.geom.envelope import Envelope

# Create RDDs
point_rdd = PointRDD(sc, "data/points.csv", 0, FileDataSplitter.CSV, True)

# Define query window
query_envelope = Envelope(-74.1, -73.9, 40.6, 40.8)

# Execute range query
result_rdd = RangeQuery.SpatialRangeQuery(
    point_rdd,
    query_envelope,
    False,  # considerBoundaryIntersection
    False   # usingIndex
)

print(f"Found {result_rdd.count()} points in range")
```

### Distance-Based Range Query

```python
# Find all points within 5km of a location
center_point = (-73.935242, 40.730610)

result = sedona.sql(f"""
    SELECT
        id,
        name,
        ST_Distance(
            ST_Transform(ST_SetSRID(geometry, 4326), 3857),
            ST_Transform(ST_SetSRID(ST_Point({center_point[0]}, {center_point[1]}), 4326), 3857)
        ) as distance_m
    FROM locations
    WHERE ST_DWithin(
        ST_Transform(ST_SetSRID(geometry, 4326), 3857),
        ST_Transform(ST_SetSRID(ST_Point({center_point[0]}, {center_point[1]}), 4326), 3857),
        5000.0
    )
    ORDER BY distance_m
""")

result.show()
```

---

## 4.3 K-Nearest Neighbors Queries (20 minutes)

### Basic KNN Query

```python
from sedona.core.spatialOperator import KNNQuery
from shapely.geometry import Point

# Reference point
query_point = Point(-73.935242, 40.730610)

# Find 10 nearest neighbors
k = 10
result = KNNQuery.SpatialKnnQuery(point_rdd, query_point, k, False)

print(f"Found {len(result)} nearest neighbors")
for i, point in enumerate(result):
    print(f"{i+1}. {point}")
```

### KNN with SQL

```python
# Register UDF for distance calculation
sedona.sql("""
    CREATE OR REPLACE TEMP VIEW locations_with_dist AS
    SELECT
        id,
        name,
        geometry,
        ST_Distance(
            ST_Transform(ST_SetSRID(geometry, 4326), 3857),
            ST_Transform(ST_SetSRID(ST_Point(-73.935242, 40.730610), 4326), 3857)
        ) as distance
    FROM locations
""")

# Get K nearest
result = sedona.sql("""
    SELECT id, name, distance
    FROM locations_with_dist
    ORDER BY distance
    LIMIT 10
""")

result.show()
```

### KNN for Each Point in Dataset

```python
# Find nearest hospital for each customer
customers.createOrReplaceTempView("customers")
hospitals.createOrReplaceTempView("hospitals")

result = sedona.sql("""
    SELECT
        c.customer_id,
        c.customer_name,
        h.hospital_id,
        h.hospital_name,
        ST_Distance(
            ST_Transform(ST_SetSRID(c.location, 4326), 3857),
            ST_Transform(ST_SetSRID(h.location, 4326), 3857)
        ) as distance_m
    FROM customers c
    CROSS JOIN hospitals h
    QUALIFY ROW_NUMBER() OVER (PARTITION BY c.customer_id ORDER BY distance_m) = 1
""")
```

**Note:** For very large datasets, use window functions cautiously as they require shuffles.

---

## 4.4 Spatial Joins (35 minutes)

### Join Types

**1. Inner Join**
- Only matching pairs
- Most common

**2. Left Join**
- Keep all from left, nulls for non-matches

**3. Right Join**
- Keep all from right, nulls for non-matches

**4. Full Outer Join**
- Keep all from both sides

### Point-in-Polygon Join (SQL)

```python
# Join customers with delivery zones
customers.createOrReplaceTempView("customers")
zones.createOrReplaceTempView("zones")

result = sedona.sql("""
    SELECT
        c.customer_id,
        c.customer_name,
        z.zone_id,
        z.zone_name
    FROM customers c
    JOIN zones z ON ST_Contains(z.zone_geom, c.location)
""")

result.show()
```

### Polygon-Polygon Join

```python
# Find overlapping parcels and zoning districts
result = sedona.sql("""
    SELECT
        p.parcel_id,
        z.zone_type,
        ST_Area(ST_Intersection(p.geometry, z.geometry)) as overlap_area,
        ST_Area(p.geometry) as parcel_area,
        (ST_Area(ST_Intersection(p.geometry, z.geometry)) / ST_Area(p.geometry)) * 100 as overlap_pct
    FROM parcels p
    JOIN zones z ON ST_Intersects(p.geometry, z.geometry)
    WHERE ST_Area(ST_Intersection(p.geometry, z.geometry)) > 0
""")
```

### Line-Polygon Join

```python
# Find which roads pass through each city
result = sedona.sql("""
    SELECT
        c.city_name,
        COUNT(DISTINCT r.road_id) as num_roads,
        SUM(ST_Length(ST_Intersection(r.geometry, c.geometry))) as total_road_length_m
    FROM cities c
    LEFT JOIN roads r ON ST_Intersects(r.geometry, c.geometry)
    GROUP BY c.city_id, c.city_name
    ORDER BY total_road_length_m DESC
""")
```

### Using RDD Join API

```python
from sedona.core.SpatialRDD import PointRDD, PolygonRDD
from sedona.core.spatialOperator import JoinQuery
from sedona.core.enums import GridType, IndexType

# Create RDDs
point_rdd = PointRDD(sc, "data/points.csv", 0, FileDataSplitter.CSV, True)
polygon_rdd = PolygonRDD(sc, "data/polygons.wkt", 0, FileDataSplitter.WKT, True)

# Spatial partition (important for performance!)
point_rdd.spatialPartitioning(GridType.KDBTREE)
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

print(f"Join result count: {result.count()}")

# Result is PairRDD: (Polygon, [Point, Point, ...])
for polygon, points in result.collect():
    print(f"Polygon contains {len(points)} points")
```

---

## 4.5 Distance Joins (20 minutes)

### Distance Join Concept
Join geometries if they're within specified distance.

### Distance Join with SQL

```python
# Find all stores within 1km of each customer
result = sedona.sql("""
    SELECT
        c.customer_id,
        s.store_id,
        s.store_name,
        ST_Distance(
            ST_Transform(ST_SetSRID(c.location, 4326), 3857),
            ST_Transform(ST_SetSRID(s.location, 4326), 3857)
        ) as distance_m
    FROM customers c
    JOIN stores s
    ON ST_DWithin(
        ST_Transform(ST_SetSRID(c.location, 4326), 3857),
        ST_Transform(ST_SetSRID(s.location, 4326), 3857),
        1000.0
    )
    ORDER BY c.customer_id, distance_m
""")
```

### Distance Join with RDD API

```python
from sedona.core.spatialOperator import JoinQuery

# Partition RDDs
object_rdd.spatialPartitioning(GridType.KDBTREE)
query_rdd.spatialPartitioning(object_rdd.getPartitioner())

# Distance join (specify distance threshold)
distance_threshold = 0.01  # degrees or appropriate units
result = JoinQuery.DistanceJoinQuery(
    object_rdd,
    query_rdd,
    distance_threshold,
    True  # useIndex
)
```

### Practical Example: Coverage Analysis

```python
# Find cell towers that cover each building
result = sedona.sql("""
    SELECT
        b.building_id,
        b.building_name,
        COUNT(t.tower_id) as num_towers_in_range,
        COLLECT_LIST(t.tower_id) as tower_ids
    FROM buildings b
    LEFT JOIN cell_towers t
    ON ST_DWithin(
        ST_Transform(ST_SetSRID(ST_Centroid(b.geometry), 4326), 3857),
        ST_Transform(ST_SetSRID(t.location, 4326), 3857),
        500.0  -- 500m coverage radius
    )
    GROUP BY b.building_id, b.building_name
    HAVING num_towers_in_range > 0
    ORDER BY num_towers_in_range DESC
""")
```

---

## 4.6 Join Optimization Strategies (15 minutes)

### Strategy 1: Broadcast Join
For small-large dataset joins.

```python
from pyspark.sql.functions import broadcast

# Broadcast small dataset (zones)
result = customers.join(
    broadcast(zones),
    expr("ST_Contains(zone_geom, location)"),
    "inner"
)
```

**When to use:**
- One dataset < 10MB
- Avoid shuffle overhead

### Strategy 2: Spatial Partitioning
Partition both datasets by spatial locality.

```python
# Partition by grid
point_rdd.spatialPartitioning(GridType.KDBTREE, 64)
polygon_rdd.spatialPartitioning(point_rdd.getPartitioner())

# Now joins are local within partitions
result = JoinQuery.SpatialJoinQuery(point_rdd, polygon_rdd, True, False)
```

### Strategy 3: Spatial Indexing
Build indexes for faster lookups.

```python
# Build R-Tree index on partitions
point_rdd.buildIndex(IndexType.RTREE, True)

# Join with index
result = JoinQuery.SpatialJoinQuery(
    point_rdd,
    polygon_rdd,
    True,   # useIndex - CRITICAL!
    False
)
```

### Strategy 4: Filter Before Join
Reduce data volume early.

```python
# Bad: Join then filter
result = sedona.sql("""
    SELECT *
    FROM large_dataset_a a
    JOIN large_dataset_b b ON ST_Intersects(a.geom, b.geom)
    WHERE a.category = 'retail'
""")

# Good: Filter then join
result = sedona.sql("""
    SELECT *
    FROM (SELECT * FROM large_dataset_a WHERE category = 'retail') a
    JOIN large_dataset_b b ON ST_Intersects(a.geom, b.geom)
""")
```

### Strategy 5: Simplify Geometries
Reduce geometric complexity.

```python
# Simplify complex polygons before join
zones_simple = sedona.sql("""
    SELECT
        zone_id,
        zone_name,
        ST_SimplifyPreserveTopology(zone_geom, 0.001) as zone_geom
    FROM zones
""")

zones_simple.createOrReplaceTempView("zones_simple")

# Now join with simplified geometries
result = sedona.sql("""
    SELECT c.customer_id, z.zone_id
    FROM customers c
    JOIN zones_simple z ON ST_Contains(z.zone_geom, c.location)
""")
```

---

## 4.7 Query Performance Tuning (10 minutes)

### Explain Query Plan

```python
# Analyze query execution plan
result.explain(True)
```

### Monitor Spark UI
- Check stage duration
- Identify skewed partitions
- Monitor shuffle reads/writes

### Optimization Checklist

**1. Partitioning**
```python
# Too few partitions: underutilization
# Too many partitions: overhead
# Sweet spot: 2-4x number of cores

df.repartition(200)  # Adjust based on data size
```

**2. Caching**
```python
# Cache frequently accessed datasets
zones.cache()
customers.cache()

# Perform operations...

# Unpersist when done
zones.unpersist()
customers.unpersist()
```

**3. Predicate Pushdown**
```python
# Enable in Spark config
spark.conf.set("spark.sql.parquet.filterPushdown", "true")
```

**4. Avoid UDFs**
```python
# Bad: Python UDF (slow)
from pyspark.sql.functions import udf
@udf("double")
def distance_udf(geom1, geom2):
    return geom1.distance(geom2)

# Good: Built-in function (fast)
df = df.withColumn("dist", expr("ST_Distance(geom1, geom2)"))
```

---

## 4.8 Sedona SQL Functions Reference (15 minutes)

### Spatial Constructors
```sql
ST_Point(x, y)
ST_MakeLine(point1, point2)
ST_MakePolygon(linestring)
ST_GeomFromWKT(wkt)
ST_GeomFromWKB(wkb)
ST_GeomFromGeoJSON(json)
```

### Spatial Predicates
```sql
ST_Contains(geom1, geom2)
ST_Intersects(geom1, geom2)
ST_Within(geom1, geom2)
ST_Overlaps(geom1, geom2)
ST_Crosses(geom1, geom2)
ST_Touches(geom1, geom2)
ST_Disjoint(geom1, geom2)
ST_Equals(geom1, geom2)
```

### Spatial Measurements
```sql
ST_Distance(geom1, geom2)
ST_Area(geom)
ST_Length(geom)
ST_Perimeter(geom)
```

### Spatial Transformations
```sql
ST_Buffer(geom, distance)
ST_Centroid(geom)
ST_Envelope(geom)
ST_ConvexHull(geom)
ST_Intersection(geom1, geom2)
ST_Union(geom1, geom2)
ST_Difference(geom1, geom2)
ST_SimplifyPreserveTopology(geom, tolerance)
```

### CRS Functions
```sql
ST_SetSRID(geom, srid)
ST_SRID(geom)
ST_Transform(geom, target_srid)
```

### Accessors
```sql
ST_X(point)
ST_Y(point)
ST_NumGeometries(geom)
ST_GeometryN(geom, n)
ST_NumPoints(geom)
ST_PointN(linestring, n)
```

---

## Hands-on Exercise 4: NYC Taxi Analysis

### Objective
Analyze NYC taxi trips using spatial joins and queries.

### Datasets
1. Taxi trip pickup/dropoff points
2. NYC borough boundaries
3. Points of interest (airports, landmarks)

### Tasks

```python
from sedona.spark import *
from pyspark.sql.functions import expr, count, avg, sum as sql_sum

# Initialize
config = SedonaContext.builder().appName('module4').getOrCreate()
sedona = SedonaContext.create(config)

# 1. Load taxi trip data
trips = sedona.read.csv("data/nyc_taxi_trips.csv", header=True)
trips = trips.withColumn("pickup_point",
    expr("ST_Transform(ST_SetSRID(ST_Point(CAST(pickup_lon AS Decimal(24,20)), CAST(pickup_lat AS Decimal(24,20))), 4326), 3857)"))
trips = trips.withColumn("dropoff_point",
    expr("ST_Transform(ST_SetSRID(ST_Point(CAST(dropoff_lon AS Decimal(24,20)), CAST(dropoff_lat AS Decimal(24,20))), 4326), 3857)"))

# 2. Load borough boundaries
boroughs = sedona.read.option("multiLine", "true").json("data/nyc_boroughs.geojson")
boroughs = boroughs.withColumn("geometry",
    expr("ST_Transform(ST_SetSRID(ST_GeomFromGeoJSON(geometry), 4326), 3857)"))

# 3. Load POIs (airports)
pois = sedona.read.csv("data/nyc_pois.csv", header=True)
pois = pois.withColumn("location",
    expr("ST_Transform(ST_SetSRID(ST_Point(CAST(lon AS Decimal(24,20)), CAST(lat AS Decimal(24,20))), 4326), 3857)"))

# Register views
trips.createOrReplaceTempView("trips")
boroughs.createOrReplaceTempView("boroughs")
pois.createOrReplaceTempView("pois")

# Task 1: Trips per borough (pickup location)
print("=== Trips per Borough ===")
result1 = sedona.sql("""
    SELECT
        b.borough_name,
        COUNT(*) as num_trips,
        AVG(CAST(t.fare_amount AS DOUBLE)) as avg_fare
    FROM trips t
    JOIN boroughs b ON ST_Contains(b.geometry, t.pickup_point)
    GROUP BY b.borough_name
    ORDER BY num_trips DESC
""")
result1.show()

# Task 2: Airport trips (within 1km of airport)
print("\n=== Airport Trips ===")
result2 = sedona.sql("""
    SELECT
        p.name as airport,
        COUNT(*) as num_pickups
    FROM pois p
    JOIN trips t ON ST_DWithin(p.location, t.pickup_point, 1000.0)
    WHERE p.type = 'airport'
    GROUP BY p.name
    ORDER BY num_pickups DESC
""")
result2.show()

# Task 3: Cross-borough trips
print("\n=== Cross-Borough Trips ===")
result3 = sedona.sql("""
    SELECT
        b1.borough_name as pickup_borough,
        b2.borough_name as dropoff_borough,
        COUNT(*) as num_trips,
        AVG(ST_Distance(t.pickup_point, t.dropoff_point)) as avg_distance_m
    FROM trips t
    JOIN boroughs b1 ON ST_Contains(b1.geometry, t.pickup_point)
    JOIN boroughs b2 ON ST_Contains(b2.geometry, t.dropoff_point)
    GROUP BY b1.borough_name, b2.borough_name
    HAVING pickup_borough != dropoff_borough
    ORDER BY num_trips DESC
    LIMIT 10
""")
result3.show()

# Task 4: Find trips near landmarks
print("\n=== Trips Near Landmarks (500m) ===")
result4 = sedona.sql("""
    SELECT
        p.name as landmark,
        COUNT(DISTINCT t.trip_id) as num_trips_nearby
    FROM pois p
    LEFT JOIN trips t ON (
        ST_DWithin(p.location, t.pickup_point, 500.0) OR
        ST_DWithin(p.location, t.dropoff_point, 500.0)
    )
    WHERE p.type = 'landmark'
    GROUP BY p.name
    ORDER BY num_trips_nearby DESC
""")
result4.show()
```

---

## Key Takeaways

- Range queries filter by spatial extent
- KNN finds nearest neighbors efficiently
- Spatial joins combine datasets by geometric relationships
- Use broadcast joins for small-large dataset combinations
- Spatial partitioning and indexing are critical for performance
- Filter data early in query pipeline
- Sedona provides comprehensive SQL spatial functions
- Always monitor Spark UI for performance bottlenecks

---

## Performance Best Practices Summary

1. Spatial partition both datasets before join
2. Build indexes on larger dataset
3. Use broadcast for small datasets (<10MB)
4. Filter early, join late
5. Simplify complex geometries
6. Cache frequently used datasets
7. Use appropriate number of partitions
8. Prefer built-in functions over UDFs

---

**Previous:** [Module 3: Spatial RDDs and Operations](../module3/README.md) | **Next:** [Module 5: Spatial Indexing and Partitioning](../module5/README.md)
