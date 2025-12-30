# Module 5: Spatial Indexing and Partitioning
**Duration:** 1.5 hours

## Overview
This module covers spatial indexing and partitioning, the key techniques that enable Sedona to process massive geospatial datasets efficiently. Understanding these concepts is crucial for production performance.

---

## 5.1 Why Spatial Indexing Matters (10 minutes)

### The Performance Problem

**Without Index:**
```
Query: Find all points in polygon
Algorithm: Check EVERY point against polygon
Complexity: O(n)
For 1 billion points: 1 billion comparisons!
```

**With Index:**
```
Query: Find all points in polygon
Algorithm: Use spatial index to narrow search
Complexity: O(log n) + O(k) where k = results
For 1 billion points: ~30 comparisons + results!
```

### Performance Impact

| Dataset Size | No Index | With R-Tree | Speedup |
|-------------|----------|-------------|---------|
| 1M geoms    | 45 sec   | 2 sec       | 22x     |
| 10M geoms   | 450 sec  | 5 sec       | 90x     |
| 100M geoms  | 4500 sec | 12 sec      | 375x    |

### When Indexing Helps Most
- Range queries
- Point-in-polygon queries
- Spatial joins
- KNN queries
- Repeated queries on same dataset

### When NOT to Index
- Single-use queries
- Very small datasets (<10K rows)
- Full table scans needed anyway
- Memory constraints

---

## 5.2 Spatial Index Types (20 minutes)

### R-Tree Index

**Concept:**
Hierarchical bounding box structure that groups nearby geometries.

```
                    Root
                   /    \
              Node1      Node2
              /   \      /   \
          Leaf1  Leaf2 Leaf3 Leaf4
          (BBs)  (BBs) (BBs)  (BBs)
```

**Characteristics:**
- Balanced tree structure
- Minimum Bounding Rectangles (MBRs) at each node
- Good for mixed query types
- Industry standard

**Best for:**
- General-purpose spatial queries
- Point-in-polygon lookups
- Range queries
- Most common choice

### Quad-Tree Index

**Concept:**
Recursively divides space into four quadrants.

```
    +--------+--------+
    |   NW   |   NE   |
    |        |        |
    +--------+--------+
    |   SW   |   SE   |
    |        |        |
    +--------+--------+
```

**Characteristics:**
- Space-based partitioning
- Fixed grid subdivision
- Simpler structure than R-Tree

**Best for:**
- Uniformly distributed data
- Point datasets
- Grid-based analysis

### Comparison

| Feature | R-Tree | Quad-Tree |
|---------|--------|-----------|
| Structure | Data-driven | Space-driven |
| Balance | Always balanced | Can be unbalanced |
| Overlap | MBRs can overlap | No overlap |
| Best for | General use | Point data |
| Build time | Slower | Faster |
| Query time | Faster | Slower |

---

## 5.3 Building Spatial Indexes (20 minutes)

### RDD API: Building Index

```python
from sedona.core.SpatialRDD import PointRDD, PolygonRDD
from sedona.core.enums import IndexType, GridType
from sedona.core.spatialOperator import JoinQuery

# Create RDD
point_rdd = PointRDD(sc, "data/points.csv", 0, FileDataSplitter.CSV, True)

# Partition first (required!)
point_rdd.spatialPartitioning(GridType.KDBTREE)

# Build R-Tree index
point_rdd.buildIndex(IndexType.RTREE, True)  # True = buildIndexOnSpatialPartitionedRDD

# Now queries will use the index automatically
polygon_rdd = PolygonRDD(sc, "data/polygons.wkt", 0, FileDataSplitter.WKT, True)
polygon_rdd.spatialPartitioning(point_rdd.getPartitioner())

# Join with index
result = JoinQuery.SpatialJoinQuery(
    point_rdd,
    polygon_rdd,
    True,   # USE INDEX!
    False
)
```

### Index Types in Sedona

```python
from sedona.core.enums import IndexType

# Available index types
IndexType.RTREE      # R-Tree (recommended)
IndexType.QUADTREE   # Quad-Tree
```

### Verification

```python
# Check if index is built
if point_rdd.indexedRDD is not None:
    print("Index built successfully")
    print(f"Index type: {point_rdd.indexedRDD}")
else:
    print("No index")
```

### Index Memory Usage

```python
# Estimate index memory
num_geometries = point_rdd.countWithoutDuplicates()
avg_geometry_size = 100  # bytes
index_overhead = 0.2     # 20% overhead

estimated_memory_mb = (num_geometries * avg_geometry_size * (1 + index_overhead)) / (1024 * 1024)
print(f"Estimated index memory: {estimated_memory_mb:.2f} MB")
```

---

## 5.4 Spatial Partitioning Strategies (25 minutes)

### Why Partition Spatially?

**Problem:**
```
Default Spark partitioning:
Partition 1: [NYC point, LA point, Tokyo point]
Partition 2: [Paris point, NYC point, London point]
Result: Cross-partition queries for nearby points!
```

**Solution:**
```
Spatial partitioning:
Partition 1: [NYC points...]
Partition 2: [LA points...]
Partition 3: [Tokyo points...]
Result: Nearby points in same partition!
```

### Grid-Based Partitioning

**Uniform Grid:**
Divide space into equal-sized cells.

```python
from sedona.core.enums import GridType

# Simple equi-grid
point_rdd.spatialPartitioning(GridType.EQUALGRID, 64)
```

**Characteristics:**
- Simple and fast
- Good for uniform data distribution
- Poor for skewed data (some partitions very full, others empty)

### KDB-Tree Partitioning

**Concept:**
K-dimensional B-tree that adapts to data distribution.

```python
# KDB-Tree partitioning (recommended)
point_rdd.spatialPartitioning(GridType.KDBTREE, 64)
```

**How it works:**
1. Recursively split data along alternating dimensions
2. Ensure balanced partitions
3. Adapt to data density

**Characteristics:**
- Adapts to data skew
- Balanced partition sizes
- Best general-purpose choice
- Slightly slower to build

### Voronoi Partitioning

```python
# Voronoi diagram partitioning
point_rdd.spatialPartitioning(GridType.VORONOI, 64)
```

**Characteristics:**
- Creates regions around sample points
- Good for clustered data
- More expensive to build

### Choosing Number of Partitions

```python
# Too few partitions: underutilization
# Too many partitions: overhead

# Rule of thumb: 2-4x number of CPU cores
num_cores = sc.defaultParallelism
num_partitions = num_cores * 3

point_rdd.spatialPartitioning(GridType.KDBTREE, num_partitions)
```

**Considerations:**
- Dataset size (larger → more partitions)
- Available memory (more partitions → less memory per partition)
- CPU cores (more cores → more partitions)
- Network bandwidth (fewer partitions → less shuffle)

### Partition Both Sides for Joins

```python
# Critical: Use same partitioner for both RDDs!

# Partition object RDD
object_rdd.spatialPartitioning(GridType.KDBTREE, 64)

# Use SAME partitioner for query RDD
query_rdd.spatialPartitioning(object_rdd.getPartitioner())

# Now join is efficient (co-located data)
result = JoinQuery.SpatialJoinQuery(object_rdd, query_rdd, True, False)
```

---

## 5.5 Partitioning and Indexing Workflow (15 minutes)

### Complete Workflow

```python
from sedona.core.SpatialRDD import PointRDD, PolygonRDD
from sedona.core.enums import IndexType, GridType, FileDataSplitter
from sedona.core.spatialOperator import JoinQuery

# 1. Load data
point_rdd = PointRDD(sc, "data/points.csv", 0, FileDataSplitter.CSV, True)
polygon_rdd = PolygonRDD(sc, "data/polygons.wkt", 0, FileDataSplitter.WKT, True)

# 2. Analyze data distribution
print(f"Points: {point_rdd.approximateTotalCount}")
print(f"Polygons: {polygon_rdd.approximateTotalCount}")
print(f"Points boundary: {point_rdd.boundaryEnvelope}")

# 3. Spatial partition (larger dataset first)
num_partitions = sc.defaultParallelism * 3
point_rdd.spatialPartitioning(GridType.KDBTREE, num_partitions)

# 4. Use same partitioner for second dataset
polygon_rdd.spatialPartitioning(point_rdd.getPartitioner())

# 5. Build index on larger dataset
point_rdd.buildIndex(IndexType.RTREE, True)

# 6. Execute join with index
result = JoinQuery.SpatialJoinQuery(
    point_rdd,
    polygon_rdd,
    True,   # useIndex
    False   # considerBoundaryIntersection
)

# 7. Process results
print(f"Join result count: {result.count()}")
```

### DataFrame API with Partitioning

```python
from sedona.sql.st_functions import ST_Transform, ST_SetSRID
from pyspark.sql.functions import expr

# Load and prepare data
df_points = sedona.read.csv("data/points.csv", header=True)
df_points = df_points.withColumn("geom",
    expr("ST_Point(CAST(lon AS Decimal(24,20)), CAST(lat AS Decimal(24,20)))"))

# Repartition by spatial hash
df_points_partitioned = df_points.repartition(
    200,  # number of partitions
    expr("ST_X(geom)"),  # partition by X coordinate
    expr("ST_Y(geom)")   # and Y coordinate
)

# Cache for reuse
df_points_partitioned.cache()

# Use in queries
df_points_partitioned.createOrReplaceTempView("points_partitioned")
```

---

## 5.6 Performance Analysis (10 minutes)

### Measuring Index Impact

```python
import time

# Without index
start = time.time()
result_no_index = JoinQuery.SpatialJoinQuery(
    point_rdd_no_index,
    polygon_rdd,
    False,  # NO index
    False
)
count_no_index = result_no_index.count()
time_no_index = time.time() - start

# With index
start = time.time()
result_with_index = JoinQuery.SpatialJoinQuery(
    point_rdd_with_index,
    polygon_rdd,
    True,   # WITH index
    False
)
count_with_index = result_with_index.count()
time_with_index = time.time() - start

print(f"No Index: {time_no_index:.2f}s")
print(f"With Index: {time_with_index:.2f}s")
print(f"Speedup: {time_no_index/time_with_index:.2f}x")
```

### Monitoring Partition Balance

```python
# Check partition sizes
partition_sizes = point_rdd.rawSpatialRDD.glom().map(len).collect()

print(f"Min partition size: {min(partition_sizes)}")
print(f"Max partition size: {max(partition_sizes)}")
print(f"Avg partition size: {sum(partition_sizes)/len(partition_sizes):.0f}")
print(f"Std deviation: {stdev(partition_sizes):.0f}")

# Good: Max/Avg ratio < 2
# Bad: Max/Avg ratio > 5 (repartition with more partitions)
```

### Spark UI Metrics
Monitor:
- Task duration per partition
- Shuffle read/write sizes
- Data skew (some tasks much slower)
- Memory usage per executor

---

## 5.7 Advanced Topics (10 minutes)

### Custom Partitioning

```python
from sedona.core.spatialPartitioning import FlatGridPartitioner

# Create custom partitioner
custom_partitioner = FlatGridPartitioner(
    point_rdd.boundaryEnvelope,
    num_partitions
)

point_rdd.spatialPartitioning(custom_partitioner)
```

### Index Serialization

```python
# Build index once, reuse
point_rdd.buildIndex(IndexType.RTREE, True)

# Save to disk (includes index)
point_rdd.rawSpatialRDD.saveAsObjectFile("data/indexed_points")

# Load with index
loaded_rdd = sc.objectFile("data/indexed_points")
```

### Multi-Level Indexes

```python
# Global index (partitioning) + Local index (per-partition)
# This is what Sedona does automatically!

# 1. Spatial partitioning = global index
point_rdd.spatialPartitioning(GridType.KDBTREE, 64)

# 2. Build local indexes on each partition
point_rdd.buildIndex(IndexType.RTREE, True)

# Result: Two-level index structure
```

---

## Hands-on Exercise 5: Optimization Challenge

### Objective
Compare performance with different partitioning and indexing strategies.

### Dataset
- 10 million taxi trip points
- NYC borough polygons

### Challenge Tasks

```python
from sedona.spark import *
from sedona.core.SpatialRDD import PointRDD, PolygonRDD
from sedona.core.enums import IndexType, GridType, FileDataSplitter
from sedona.core.spatialOperator import JoinQuery
import time

# Initialize
config = SedonaContext.builder().appName('module5').getOrCreate()
sedona = SedonaContext.create(config)
sc = sedona.sparkContext

# Load data
points_rdd = PointRDD(sc, "data/taxi_points.csv", 0, FileDataSplitter.CSV, True)
boroughs_rdd = PolygonRDD(sc, "data/boroughs.wkt", 0, FileDataSplitter.WKT, True)

print(f"Points: {points_rdd.approximateTotalCount:,}")
print(f"Boroughs: {boroughs_rdd.approximateTotalCount}")

# Task 1: Baseline (no partitioning, no index)
print("\n=== Baseline: No optimization ===")
start = time.time()
result1 = JoinQuery.SpatialJoinQuery(points_rdd, boroughs_rdd, False, False)
count1 = result1.count()
time1 = time.time() - start
print(f"Time: {time1:.2f}s, Results: {count1:,}")

# Task 2: With partitioning only
print("\n=== With spatial partitioning ===")
points_rdd2 = PointRDD(sc, "data/taxi_points.csv", 0, FileDataSplitter.CSV, True)
points_rdd2.spatialPartitioning(GridType.KDBTREE, 64)
boroughs_rdd2 = PolygonRDD(sc, "data/boroughs.wkt", 0, FileDataSplitter.WKT, True)
boroughs_rdd2.spatialPartitioning(points_rdd2.getPartitioner())

start = time.time()
result2 = JoinQuery.SpatialJoinQuery(points_rdd2, boroughs_rdd2, False, False)
count2 = result2.count()
time2 = time.time() - start
print(f"Time: {time2:.2f}s, Results: {count2:,}")
print(f"Speedup: {time1/time2:.2f}x")

# Task 3: With partitioning AND indexing
print("\n=== With partitioning + R-Tree index ===")
points_rdd3 = PointRDD(sc, "data/taxi_points.csv", 0, FileDataSplitter.CSV, True)
points_rdd3.spatialPartitioning(GridType.KDBTREE, 64)
points_rdd3.buildIndex(IndexType.RTREE, True)
boroughs_rdd3 = PolygonRDD(sc, "data/boroughs.wkt", 0, FileDataSplitter.WKT, True)
boroughs_rdd3.spatialPartitioning(points_rdd3.getPartitioner())

start = time.time()
result3 = JoinQuery.SpatialJoinQuery(points_rdd3, boroughs_rdd3, True, False)
count3 = result3.count()
time3 = time.time() - start
print(f"Time: {time3:.2f}s, Results: {count3:,}")
print(f"Speedup vs baseline: {time1/time3:.2f}x")
print(f"Speedup vs partitioned only: {time2/time3:.2f}x")

# Task 4: Compare partition strategies
print("\n=== Comparing partition strategies ===")

strategies = [
    (GridType.EQUALGRID, "Equal Grid"),
    (GridType.KDBTREE, "KDB-Tree"),
]

for grid_type, name in strategies:
    points_rdd_temp = PointRDD(sc, "data/taxi_points.csv", 0, FileDataSplitter.CSV, True)
    points_rdd_temp.spatialPartitioning(grid_type, 64)
    points_rdd_temp.buildIndex(IndexType.RTREE, True)

    boroughs_rdd_temp = PolygonRDD(sc, "data/boroughs.wkt", 0, FileDataSplitter.WKT, True)
    boroughs_rdd_temp.spatialPartitioning(points_rdd_temp.getPartitioner())

    start = time.time()
    result_temp = JoinQuery.SpatialJoinQuery(points_rdd_temp, boroughs_rdd_temp, True, False)
    count_temp = result_temp.count()
    time_temp = time.time() - start

    print(f"{name}: {time_temp:.2f}s")

# Task 5: Analyze partition balance
print("\n=== Partition Balance Analysis ===")
partition_sizes = points_rdd3.rawSpatialRDD.glom().map(len).collect()
print(f"Number of partitions: {len(partition_sizes)}")
print(f"Min size: {min(partition_sizes):,}")
print(f"Max size: {max(partition_sizes):,}")
print(f"Avg size: {sum(partition_sizes)/len(partition_sizes):,.0f}")
print(f"Max/Avg ratio: {max(partition_sizes)/(sum(partition_sizes)/len(partition_sizes)):.2f}")
```

### Expected Results
- Baseline: Slowest
- With partitioning: 2-5x faster
- With partitioning + index: 10-100x faster
- KDB-Tree: Better balance than equal grid

---

## Key Takeaways

- Spatial indexing provides dramatic performance improvements (10-100x)
- R-Tree is the recommended index type for most use cases
- Spatial partitioning is essential for distributed processing
- KDB-Tree partitioning adapts to data distribution
- Always partition both datasets with same partitioner for joins
- Build indexes AFTER partitioning
- Monitor partition balance to avoid data skew
- Index overhead: ~20% additional memory

---

## Optimization Decision Tree

```
Is dataset > 1M geometries?
  No → Don't bother with indexing
  Yes ↓

Will you run multiple queries?
  No → May not be worth index build cost
  Yes ↓

Partition strategy:
  - Uniform data → EQUALGRID
  - Skewed data → KDBTREE (recommended)
  - Very skewed → Increase partitions

Index type:
  - General use → RTREE (recommended)
  - Point data, simple queries → QUADTREE

Number of partitions:
  - Start with: 3 × number of cores
  - Adjust based on partition size balance
```

---

## Quiz Questions

1. What is the time complexity improvement from indexing?
2. What's the difference between R-Tree and Quad-Tree?
3. Why must you partition before building an index?
4. How do you ensure both RDDs use the same partitioner?
5. What is a good Max/Avg partition size ratio?
6. When should you NOT use spatial indexing?

---

**Previous:** [Module 4: Spatial Queries and Joins](../module4/README.md) | **Next:** [Module 6: Advanced Analytics and Visualization](../module6/README.md)
