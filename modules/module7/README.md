# Module 7: Real-world Case Studies and Best Practices
**Duration:** 1 hour

## Overview
This module presents real-world applications of Apache Sedona, production deployment strategies, performance optimization techniques, and best practices learned from industry use cases.

---

## 7.1 Case Study 1: Ride-Sharing Zone Analysis (15 minutes)

### Business Problem
A ride-sharing company needs to:
- Analyze 100M+ daily trip records
- Optimize driver dispatch zones
- Identify high-demand areas in real-time
- Reduce customer wait times

### Architecture

```
┌─────────────────────────────────────────┐
│  Real-time Trip Stream (Kafka)          │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│  Spark Streaming + Sedona               │
│  - Parse GPS coordinates                │
│  - Spatial join with zones              │
│  - Aggregate demand by zone             │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│  Real-time Dashboard                    │
│  - Heatmap visualization                │
│  - Demand predictions                   │
│  - Driver allocation                    │
└─────────────────────────────────────────┘
```

### Solution

```python
from sedona.spark import *
from pyspark.sql.functions import window, count, avg

# Initialize Sedona with streaming
config = SedonaContext.builder() \
    .config("spark.sql.streaming.checkpointLocation", "/tmp/checkpoint") \
    .appName('rideshare-analysis') \
    .getOrCreate()
sedona = SedonaContext.create(config)

# Load dispatch zones (static)
zones = sedona.read.format("geoparquet").load("data/dispatch_zones.parquet")
zones = zones.selectExpr(
    "zone_id",
    "zone_name",
    "ST_Transform(ST_SetSRID(geometry, 4326), 3857) as zone_geom"
)
zones.cache()  # Static reference data

# Stream trip requests
trip_stream = sedona.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "trip_requests") \
    .load()

# Parse and create geometries
trips = trip_stream.selectExpr(
    "CAST(value AS STRING) as json"
).selectExpr(
    "get_json_object(json, '$.trip_id') as trip_id",
    "CAST(get_json_object(json, '$.timestamp') AS TIMESTAMP) as timestamp",
    "CAST(get_json_object(json, '$.pickup_lon') AS DOUBLE) as lon",
    "CAST(get_json_object(json, '$.pickup_lat') AS DOUBLE) as lat"
).withColumn("pickup_point",
    expr("ST_Transform(ST_SetSRID(ST_Point(lon, lat), 4326), 3857)")
)

# Spatial join with zones
trips_zoned = trips.join(
    zones,
    expr("ST_Contains(zone_geom, pickup_point)")
)

# Aggregate demand by 5-minute windows
demand = trips_zoned.groupBy(
    window("timestamp", "5 minutes"),
    "zone_id",
    "zone_name"
).agg(
    count("*").alias("trip_count")
)

# Write to dashboard
query = demand.writeStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("topic", "zone_demand") \
    .outputMode("update") \
    .start()

query.awaitTermination()
```

### Results
- **Processing rate**: 50K events/second
- **Latency**: <2 seconds end-to-end
- **Cost reduction**: 40% fewer empty miles
- **Customer satisfaction**: +15% from reduced wait times

### Key Learnings
1. Cache static reference data (zones)
2. Transform to projected CRS once for distance calculations
3. Use broadcast join for small zone datasets
4. Checkpoint streaming state regularly
5. Monitor partition skew in high-density zones

---

## 7.2 Case Study 2: Urban Heat Island Detection (15 minutes)

### Business Problem
City planning department needs to:
- Identify urban heat islands from satellite imagery
- Correlate with building density and vegetation
- Prioritize green infrastructure investments
- Monitor changes over time

### Data Sources
- Landsat thermal imagery (raster)
- Building footprints (vector polygons)
- Tree canopy data (vector polygons)
- Census blocks (vector polygons)

### Solution

```python
from sedona.spark import *
from pyspark.sql.functions import expr, avg, sum as sql_sum

# Load raster thermal data (converted to points)
thermal_points = sedona.read.format("geoparquet") \
    .load("data/thermal_pixels.parquet")

# Load building footprints
buildings = sedona.read.format("geoparquet") \
    .load("data/buildings.parquet")

# Load tree canopy
trees = sedona.read.format("geoparquet") \
    .load("data/tree_canopy.parquet")

# Load census blocks
census_blocks = sedona.read.format("geoparquet") \
    .load("data/census_blocks.parquet")

# Register views
thermal_points.createOrReplaceTempView("thermal")
buildings.createOrReplaceTempView("buildings")
trees.createOrReplaceTempView("trees")
census_blocks.createOrReplaceTempView("census")

# Create analysis grid (500m cells)
grid_size = 500  # meters

analysis = sedona.sql(f"""
    WITH grid AS (
        SELECT
            FLOOR(ST_X(geom) / {grid_size}) * {grid_size} as grid_x,
            FLOOR(ST_Y(geom) / {grid_size}) * {grid_size} as grid_y,
            ST_MakeEnvelope(
                FLOOR(ST_X(geom) / {grid_size}) * {grid_size},
                FLOOR(ST_Y(geom) / {grid_size}) * {grid_size},
                (FLOOR(ST_X(geom) / {grid_size}) + 1) * {grid_size},
                (FLOOR(ST_Y(geom) / {grid_size}) + 1) * {grid_size}
            ) as cell_geom
        FROM thermal
        GROUP BY grid_x, grid_y
    ),
    grid_metrics AS (
        SELECT
            g.grid_x,
            g.grid_y,
            g.cell_geom,
            AVG(t.temperature) as avg_temp,
            SUM(CASE WHEN ST_Intersects(b.geometry, g.cell_geom)
                THEN ST_Area(ST_Intersection(b.geometry, g.cell_geom))
                ELSE 0 END) as building_area,
            SUM(CASE WHEN ST_Intersects(tr.geometry, g.cell_geom)
                THEN ST_Area(ST_Intersection(tr.geometry, g.cell_geom))
                ELSE 0 END) as tree_area
        FROM grid g
        LEFT JOIN thermal t ON ST_Contains(g.cell_geom, t.geom)
        LEFT JOIN buildings b ON ST_Intersects(b.geometry, g.cell_geom)
        LEFT JOIN trees tr ON ST_Intersects(tr.geometry, g.cell_geom)
        GROUP BY g.grid_x, g.grid_y, g.cell_geom
    )
    SELECT
        grid_x,
        grid_y,
        avg_temp,
        building_area,
        tree_area,
        ({grid_size} * {grid_size}) as cell_area,
        (building_area / ({grid_size} * {grid_size})) * 100 as building_pct,
        (tree_area / ({grid_size} * {grid_size})) * 100 as tree_pct,
        CASE
            WHEN avg_temp > 35 AND tree_pct < 20 THEN 'High Priority'
            WHEN avg_temp > 32 AND tree_pct < 30 THEN 'Medium Priority'
            ELSE 'Low Priority'
        END as intervention_priority
    FROM grid_metrics
    WHERE avg_temp IS NOT NULL
    ORDER BY avg_temp DESC
""")

# Export results
analysis.write.format("geoparquet") \
    .mode("overwrite") \
    .save("output/heat_island_analysis.parquet")

# Summary statistics
print("=== Heat Island Analysis Summary ===")
summary = analysis.groupBy("intervention_priority").count()
summary.show()
```

### Results
- Identified 150+ heat island zones
- 80% correlation between low tree cover and high temperature
- Prioritized $50M in green infrastructure spending
- Projected 2-3°C temperature reduction in target areas

---

## 7.3 Case Study 3: Retail Location Intelligence (15 minutes)

### Business Problem
Retail chain needs to:
- Evaluate 1000+ potential store locations
- Analyze competitor proximity
- Estimate customer catchment areas
- Optimize store placement

### Solution

```python
from sedona.spark import *
from pyspark.sql.functions import expr, count, sum as sql_sum

# Load candidate locations
candidates = sedona.read.csv("data/candidate_locations.csv", header=True)
candidates = candidates.withColumn("location",
    expr("ST_Transform(ST_SetSRID(ST_Point(CAST(lon AS Decimal(24,20)), CAST(lat AS Decimal(24,20))), 4326), 3857)"))

# Load existing stores (ours + competitors)
stores = sedona.read.csv("data/all_stores.csv", header=True)
stores = stores.withColumn("location",
    expr("ST_Transform(ST_SetSRID(ST_Point(CAST(lon AS Decimal(24,20)), CAST(lat AS Decimal(24,20))), 4326), 3857)"))

# Load demographics by census block
demographics = sedona.read.format("geoparquet") \
    .load("data/demographics.parquet")

# Load residential density
residential = sedona.read.format("geoparquet") \
    .load("data/residential_buildings.parquet")

# Register views
candidates.createOrReplaceTempView("candidates")
stores.createOrReplaceTempView("stores")
demographics.createOrReplaceTempView("demographics")
residential.createOrReplaceTempView("residential")

# Analyze each candidate location
analysis = sedona.sql("""
    WITH candidate_buffers AS (
        SELECT
            candidate_id,
            location,
            ST_Buffer(location, 1000) as buffer_1km,
            ST_Buffer(location, 3000) as buffer_3km
        FROM candidates
    ),
    competitor_analysis AS (
        SELECT
            cb.candidate_id,
            COUNT(CASE WHEN s.brand = 'competitor_a' THEN 1 END) as competitors_a_1km,
            COUNT(CASE WHEN s.brand = 'competitor_b' THEN 1 END) as competitors_b_1km,
            COUNT(CASE WHEN s.brand = 'our_brand' THEN 1 END) as our_stores_3km,
            MIN(CASE WHEN s.brand != 'our_brand'
                THEN ST_Distance(cb.location, s.location)
                ELSE NULL END) as nearest_competitor_m
        FROM candidate_buffers cb
        LEFT JOIN stores s ON (
            (s.brand LIKE 'competitor%' AND ST_DWithin(cb.location, s.location, 1000)) OR
            (s.brand = 'our_brand' AND ST_DWithin(cb.location, s.location, 3000))
        )
        GROUP BY cb.candidate_id
    ),
    demographic_analysis AS (
        SELECT
            cb.candidate_id,
            SUM(d.population) as population_3km,
            AVG(d.median_income) as avg_income,
            SUM(d.households) as households_3km
        FROM candidate_buffers cb
        LEFT JOIN demographics d ON ST_Intersects(cb.buffer_3km, d.geometry)
        GROUP BY cb.candidate_id
    ),
    residential_analysis AS (
        SELECT
            cb.candidate_id,
            COUNT(r.building_id) as residential_units_1km
        FROM candidate_buffers cb
        LEFT JOIN residential r ON ST_Contains(cb.buffer_1km, r.geometry)
        GROUP BY cb.candidate_id
    )
    SELECT
        c.candidate_id,
        c.address,
        ca.competitors_a_1km,
        ca.competitors_b_1km,
        ca.our_stores_3km,
        ca.nearest_competitor_m,
        da.population_3km,
        da.avg_income,
        da.households_3km,
        ra.residential_units_1km,
        -- Scoring model
        (
            (da.population_3km / 10000.0) * 0.3 +
            (da.avg_income / 100000.0) * 0.2 +
            (ra.residential_units_1km / 1000.0) * 0.2 +
            (CASE WHEN ca.nearest_competitor_m > 500 THEN 1.0 ELSE 0.5 END) * 0.15 +
            (CASE WHEN ca.our_stores_3km = 0 THEN 1.0 ELSE 0.3 END) * 0.15
        ) * 100 as location_score
    FROM candidates c
    LEFT JOIN competitor_analysis ca ON c.candidate_id = ca.candidate_id
    LEFT JOIN demographic_analysis da ON c.candidate_id = da.candidate_id
    LEFT JOIN residential_analysis ra ON c.candidate_id = ra.candidate_id
    ORDER BY location_score DESC
""")

# Show top locations
print("=== Top 20 Candidate Locations ===")
analysis.show(20, truncate=False)

# Export results
analysis.write.format("geoparquet") \
    .mode("overwrite") \
    .save("output/location_analysis.parquet")
```

### Results
- Evaluated 1000 locations in 15 minutes
- Identified top 50 locations with 85+ scores
- Opened 12 stores in first wave
- 20% higher revenue than chain average
- ROI positive within 8 months

---

## 7.4 Performance Optimization Best Practices (15 minutes)

### 1. Data Format Optimization

**Use GeoParquet for Big Data:**
```python
# DON'T: Use Shapefile for large datasets
# Shapefile has 2GB file limit, slow to read

# DO: Use GeoParquet
df.write.format("geoparquet") \
    .option("compression", "snappy") \
    .save("output/data.parquet")

# Partition by spatial attributes
df.write.format("geoparquet") \
    .partitionBy("region", "year") \
    .save("output/partitioned_data.parquet")
```

### 2. Query Optimization

**Push Down Filters:**
```python
# DON'T: Filter after expensive operation
result = df.join(other_df, condition).filter("category = 'A'")

# DO: Filter before join
result = df.filter("category = 'A'").join(other_df, condition)
```

**Use Broadcast for Small Tables:**
```python
from pyspark.sql.functions import broadcast

# Broadcast small dataset (<10MB)
result = large_df.join(broadcast(small_df), condition)
```

### 3. Geometry Simplification

```python
# Simplify complex geometries before operations
df = df.withColumn("simple_geom",
    expr("ST_SimplifyPreserveTopology(geometry, 0.001)"))

# Use simplified geometry for joins
result = df.join(other_df,
    expr("ST_Intersects(simple_geom, other_geom)"))
```

### 4. Partitioning Strategy

```python
# Partition by spatial attributes
df.repartition(200, "grid_x", "grid_y")

# For RDDs: use spatial partitioning
from sedona.core.enums import GridType
spatial_rdd.spatialPartitioning(GridType.KDBTREE, 200)
```

### 5. Caching Strategy

```python
# Cache static reference data
zones.cache()

# Unpersist when no longer needed
zones.unpersist()

# Check cache usage
spark.sparkContext.statusTracker().getExecutorInfos()
```

### 6. Avoid Expensive Operations

```python
# DON'T: Use ST_Union on millions of geometries
result = df.selectExpr("ST_Union_Aggr(geometry)")

# DO: Hierarchical aggregation
result = df.groupBy("region") \
    .agg(expr("ST_Union_Aggr(geometry) as region_geom")) \
    .selectExpr("ST_Union_Aggr(region_geom)")
```

### 7. Monitor and Tune

```bash
# Spark UI: localhost:4040
# Monitor:
# - Stage durations
# - Shuffle read/write
# - Task skew
# - Memory usage
# - GC time
```

**Key Metrics:**
- Stage duration: Identify bottlenecks
- Shuffle size: Reduce with better partitioning
- Task time variance: Indicates data skew
- GC time: Should be <10% of task time

---

## 7.5 Production Deployment Strategies (5 minutes)

### Cluster Configuration

```python
# Recommended Spark configuration for Sedona
config = {
    "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
    "spark.kryo.registrator": "org.apache.sedona.core.serde.SedonaKryoRegistrator",
    "spark.sql.adaptive.enabled": "true",
    "spark.sql.adaptive.coalescePartitions.enabled": "true",
    "spark.sql.adaptive.skewJoin.enabled": "true",
    "spark.executor.memory": "16g",
    "spark.executor.cores": "4",
    "spark.driver.memory": "8g",
    "spark.default.parallelism": "400",
    "spark.sql.shuffle.partitions": "400"
}
```

### Resource Planning

**Rule of Thumb:**
- **Executor memory**: 16-32 GB per executor
- **Executor cores**: 4-6 cores per executor
- **Partitions**: 2-4x total cores
- **Driver memory**: 1/4 of executor memory

### Error Handling

```python
try:
    result = sedona.sql("""
        SELECT ST_Area(geometry) as area
        FROM complex_geometries
    """)
except Exception as e:
    logger.error(f"Query failed: {str(e)}")
    # Fallback or retry logic
```

### Data Quality Validation

```python
# Validate geometries
df = df.filter(expr("ST_IsValid(geometry)"))

# Fix invalid geometries
df = df.withColumn("geometry",
    expr("ST_MakeValid(geometry)"))

# Check for nulls
df = df.filter(expr("geometry IS NOT NULL"))
```

---

## 7.6 Common Pitfalls and Solutions (5 minutes)

### Pitfall 1: Mixing CRS
**Problem:** Joining datasets with different coordinate systems.

**Solution:**
```python
# Always transform to common CRS
df1 = df1.withColumn("geom", expr("ST_Transform(ST_SetSRID(geom, 4326), 3857)"))
df2 = df2.withColumn("geom", expr("ST_Transform(ST_SetSRID(geom, 4326), 3857)"))
```

### Pitfall 2: Forgetting to Set SRID
**Problem:** Distance calculations fail or give wrong results.

**Solution:**
```python
# Always set SRID
df = df.withColumn("geom", expr("ST_SetSRID(geom, 4326)"))
```

### Pitfall 3: Not Partitioning Before Join
**Problem:** Spatial joins take hours.

**Solution:**
```python
# Partition both RDDs
rdd1.spatialPartitioning(GridType.KDBTREE, 200)
rdd2.spatialPartitioning(rdd1.getPartitioner())
rdd1.buildIndex(IndexType.RTREE, True)
```

### Pitfall 4: Using Wrong Data Types
**Problem:** Coordinates stored as strings.

**Solution:**
```python
# Cast to Decimal (not Double, to avoid precision loss)
df = df.withColumn("lon", expr("CAST(lon AS Decimal(24,20))"))
df = df.withColumn("lat", expr("CAST(lat AS Decimal(24,20))"))
```

### Pitfall 5: Memory Issues with Large Geometries
**Problem:** OOM errors with complex polygons.

**Solution:**
```python
# Simplify geometries
df = df.withColumn("geom", expr("ST_SimplifyPreserveTopology(geom, 0.001)"))

# Or increase memory
spark.conf.set("spark.executor.memory", "32g")
```

---

## Key Takeaways

- Real-world applications require combining spatial and non-spatial analysis
- Performance optimization is critical for production deployments
- GeoParquet is the recommended format for big geospatial data
- Always partition spatially before joins
- Monitor Spark UI for performance bottlenecks
- Validate and fix geometries before processing
- Cache static reference data
- Transform to projected CRS for distance operations
- Use broadcast joins for small datasets
- Test with sample data before scaling to full datasets

---

## Production Checklist

- [ ] Validate all geometries with ST_IsValid
- [ ] Set SRID on all geometries
- [ ] Transform to common CRS before operations
- [ ] Partition spatially for large datasets
- [ ] Build indexes for repeated queries
- [ ] Cache static reference data
- [ ] Configure Kryo serialization
- [ ] Enable adaptive query execution
- [ ] Monitor Spark UI metrics
- [ ] Implement error handling
- [ ] Set up checkpointing for streaming
- [ ] Use GeoParquet for storage
- [ ] Test query plans with EXPLAIN
- [ ] Validate output data quality
- [ ] Document CRS and data sources

---

## Additional Resources

- [Apache Sedona Production Guide](https://sedona.apache.org/latest-snapshot/setup/production/)
- [Performance Tuning](https://sedona.apache.org/latest-snapshot/tutorial/rdd/#performance-tuning)
- [Spark Performance Tuning Guide](https://spark.apache.org/docs/latest/tuning.html)
- [GeoParquet Specification](https://geoparquet.org/)

---

**Previous:** [Module 6: Advanced Analytics and Visualization](../module6/README.md) | **Next:** [Final Project](../../exercises/README.md)
