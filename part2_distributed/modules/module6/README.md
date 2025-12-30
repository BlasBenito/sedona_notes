# Module 6: Advanced Analytics and Visualization
**Duration:** 1.5 hours

## Overview
This module explores advanced spatial analytics capabilities including aggregations, clustering, raster processing, visualization, and streaming geospatial data.

---

## 6.1 Spatial Aggregations (20 minutes)

### Group By with Spatial Operations

```python
# Count points per polygon
result = sedona.sql("""
    SELECT
        z.zone_id,
        z.zone_name,
        COUNT(*) as point_count,
        ST_Union_Aggr(p.geometry) as all_points
    FROM zones z
    LEFT JOIN points p ON ST_Contains(z.geometry, p.geometry)
    GROUP BY z.zone_id, z.zone_name
""")
```

### Spatial Union Aggregation

```python
# Merge all geometries in a group
result = sedona.sql("""
    SELECT
        county,
        ST_Union_Aggr(geometry) as county_boundary
    FROM parcels
    GROUP BY county
""")
```

### Envelope Aggregation

```python
# Get bounding box of all geometries in group
result = sedona.sql("""
    SELECT
        region,
        ST_Envelope_Aggr(geometry) as region_bbox
    FROM locations
    GROUP BY region
""")
```

### Statistical Aggregations

```python
# Spatial statistics per group
result = sedona.sql("""
    SELECT
        zone_id,
        COUNT(*) as num_buildings,
        SUM(ST_Area(geometry)) as total_area,
        AVG(ST_Area(geometry)) as avg_area,
        MIN(ST_Area(geometry)) as min_area,
        MAX(ST_Area(geometry)) as max_area,
        STDDEV(ST_Area(geometry)) as stddev_area
    FROM buildings
    GROUP BY zone_id
    ORDER BY total_area DESC
""")
```

### Centroid of Points

```python
# Find geometric center of point cluster
result = sedona.sql("""
    SELECT
        cluster_id,
        ST_Centroid(ST_Union_Aggr(geometry)) as cluster_center,
        COUNT(*) as num_points
    FROM clustered_points
    GROUP BY cluster_id
""")
```

---

## 6.2 Heatmap Generation (15 minutes)

### Concept
Heatmaps visualize point density across geographic areas.

### Grid-Based Heatmap

```python
from pyspark.sql.functions import expr, floor

# Create grid cells
grid_size = 0.01  # degrees

df_heatmap = sedona.sql(f"""
    SELECT
        FLOOR(ST_X(geometry) / {grid_size}) * {grid_size} as grid_x,
        FLOOR(ST_Y(geometry) / {grid_size}) * {grid_size} as grid_y,
        COUNT(*) as point_count
    FROM points
    GROUP BY grid_x, grid_y
    HAVING point_count > 0
""")

# Add geometry for visualization
df_heatmap = df_heatmap.withColumn("cell_geom",
    expr(f"ST_MakeEnvelope(grid_x, grid_y, grid_x + {grid_size}, grid_y + {grid_size})"))

df_heatmap.show()
```

### Hexagon Grid Heatmap

```python
# H3 hexagonal grid (requires h3-py)
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType
import h3

@udf(StringType())
def lat_lon_to_h3(lat, lon, resolution=9):
    return h3.geo_to_h3(lat, lon, resolution)

# Create hexagon heatmap
df_hex = df_points.withColumn("h3_index",
    lat_lon_to_h3(expr("ST_Y(geometry)"), expr("ST_X(geometry)")))

df_heatmap_hex = df_hex.groupBy("h3_index").count()
df_heatmap_hex.show()
```

### Kernel Density Estimation

```python
# Simplified KDE using buffer union
result = sedona.sql("""
    WITH buffered AS (
        SELECT ST_Buffer(geometry, 0.01) as buffer_geom
        FROM points
    ),
    grid AS (
        SELECT
            (x * 0.01) as grid_x,
            (y * 0.01) as grid_y,
            ST_Point(x * 0.01, y * 0.01) as grid_point
        FROM (SELECT sequence(-180, 180, 1) as x) xs
        CROSS JOIN (SELECT sequence(-90, 90, 1) as y) ys
    )
    SELECT
        grid_x,
        grid_y,
        COUNT(b.buffer_geom) as density
    FROM grid g
    LEFT JOIN buffered b ON ST_Contains(b.buffer_geom, g.grid_point)
    GROUP BY grid_x, grid_y
    HAVING density > 0
""")
```

---

## 6.3 Spatial Clustering (25 minutes)

### K-Means Clustering with Spatial Features

```python
from pyspark.ml.clustering import KMeans
from pyspark.ml.feature import VectorAssembler
from pyspark.sql.functions import expr

# Extract coordinates
df_coords = df_points.withColumn("x", expr("ST_X(geometry)")) \
                      .withColumn("y", expr("ST_Y(geometry)"))

# Create feature vector
assembler = VectorAssembler(inputCols=["x", "y"], outputCol="features")
df_features = assembler.transform(df_coords)

# K-Means clustering
kmeans = KMeans(k=10, seed=42)
model = kmeans.fit(df_features)

# Predict clusters
df_clustered = model.transform(df_features)
df_clustered.select("id", "prediction").show()

# Get cluster centers
centers = model.clusterCenters()
for i, center in enumerate(centers):
    print(f"Cluster {i}: ({center[0]:.4f}, {center[1]:.4f})")
```

### DBSCAN Clustering (Density-Based)

```python
# Approximate DBSCAN using self-join
epsilon = 0.01  # distance threshold
min_points = 5

# Find neighbors for each point
result = sedona.sql(f"""
    WITH neighbors AS (
        SELECT
            p1.id as point_id,
            COUNT(p2.id) as neighbor_count,
            COLLECT_LIST(p2.id) as neighbors
        FROM points p1
        JOIN points p2 ON ST_Distance(p1.geometry, p2.geometry) <= {epsilon}
        WHERE p1.id != p2.id
        GROUP BY p1.id
    )
    SELECT
        point_id,
        neighbor_count,
        CASE
            WHEN neighbor_count >= {min_points} THEN 'core'
            WHEN neighbor_count > 0 THEN 'border'
            ELSE 'noise'
        END as point_type
    FROM neighbors
""")

result.groupBy("point_type").count().show()
```

### Spatial Clustering with Constraints

```python
# Cluster points within regions
result = sedona.sql("""
    WITH point_regions AS (
        SELECT
            p.id as point_id,
            p.geometry,
            r.region_id
        FROM points p
        JOIN regions r ON ST_Contains(r.geometry, p.geometry)
    )
    SELECT
        region_id,
        COUNT(*) as points_in_region,
        ST_Centroid(ST_Union_Aggr(geometry)) as region_center
    FROM point_regions
    GROUP BY region_id
""")
```

---

## 6.4 Raster Data Processing (20 minutes)

### Loading Raster Data

```python
from sedona.core.formatMapper.RasterReader import RasterReader

# Read GeoTIFF raster
raster_rdd = RasterReader.readGeoTiffToSpatialRDD(sc, "data/elevation.tif")

print(f"Raster count: {raster_rdd.count()}")
```

### Raster Operations

```python
# Extract raster values at points
from sedona.core.spatialOperator import RasterQuery

# Query raster at point locations
result = RasterQuery.spatialJoin(point_rdd, raster_rdd)
```

### Rasterize Vector Data

```python
# Convert polygons to raster
from sedona.sql.st_functions import RS_Rasterize

result = sedona.sql("""
    SELECT RS_Rasterize(
        geometry,
        1000,  -- width
        1000,  -- height
        attribute_value
    ) as raster
    FROM polygons
""")
```

### Raster Statistics

```python
# Compute zonal statistics
result = sedona.sql("""
    WITH raster_points AS (
        SELECT
            pixel_x,
            pixel_y,
            pixel_value,
            ST_Point(pixel_x, pixel_y) as pixel_geom
        FROM raster_data
    )
    SELECT
        z.zone_id,
        COUNT(*) as pixel_count,
        AVG(rp.pixel_value) as mean_value,
        MIN(rp.pixel_value) as min_value,
        MAX(rp.pixel_value) as max_value
    FROM zones z
    LEFT JOIN raster_points rp ON ST_Contains(z.geometry, rp.pixel_geom)
    GROUP BY z.zone_id
""")
```

---

## 6.5 Map Visualization (20 minutes)

### Using Sedona-Viz (Deprecated but useful)

```python
from sedona.viz import SedonaVizRegistrator
from sedona.viz.core.SedonaViz import SedonaViz

# Register viz functions
SedonaVizRegistrator.registerAll(sedona)

# Create visualization
viz = SedonaViz(sedona, 1000, 800)  # width, height

# Add point layer
viz.add_layer(point_rdd, fill_color="red", stroke_width=1)

# Add polygon layer
viz.add_layer(polygon_rdd, fill_color="blue", stroke_color="black", opacity=0.3)

# Save as image
viz.save("output/map.png")
```

### Export for Web Visualization

**Kepler.gl Format:**
```python
# Export to GeoJSON for Kepler.gl
df_export = sedona.sql("""
    SELECT
        id,
        name,
        ST_AsGeoJSON(geometry) as geometry,
        attribute1,
        attribute2
    FROM spatial_data
""")

# Save as GeoJSON
df_export.coalesce(1).write.mode("overwrite").json("output/kepler_data.geojson")
```

**DeckGL Format:**
```python
# Export for Deck.gl
df_export = df_points.selectExpr(
    "id",
    "ST_X(geometry) as longitude",
    "ST_Y(geometry) as latitude",
    "value",
    "category"
)

df_export.write.mode("overwrite").json("output/deckgl_data.json")
```

### Creating Interactive Maps with Python

```python
# Using folium for visualization
import folium
from folium.plugins import HeatMap

# Sample data for visualization
sample_data = df_points.limit(1000).toPandas()

# Create base map
m = folium.Map(location=[40.7, -74.0], zoom_start=11)

# Add points
for idx, row in sample_data.iterrows():
    folium.CircleMarker(
        location=[row['latitude'], row['longitude']],
        radius=3,
        color='red',
        fill=True
    ).add_to(m)

# Save map
m.save("output/interactive_map.html")
```

### Heatmap Visualization

```python
# Create heatmap
m_heat = folium.Map(location=[40.7, -74.0], zoom_start=11)

heat_data = [[row['latitude'], row['longitude'], row['value']]
             for idx, row in sample_data.iterrows()]

HeatMap(heat_data).add_to(m_heat)
m_heat.save("output/heatmap.html")
```

---

## 6.6 Streaming Geospatial Data (10 minutes)

### Structured Streaming with Spatial Operations

```python
from pyspark.sql.functions import window, expr
from sedona.sql import st_functions as ST

# Define streaming source
streaming_df = sedona.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "geo_events") \
    .load()

# Parse JSON and create geometries
parsed_df = streaming_df.select(
    expr("CAST(value AS STRING)").alias("json_data")
).select(
    expr("get_json_object(json_data, '$.id')").alias("event_id"),
    expr("get_json_object(json_data, '$.timestamp')").alias("timestamp"),
    expr("CAST(get_json_object(json_data, '$.lon') AS DOUBLE)").alias("lon"),
    expr("CAST(get_json_object(json_data, '$.lat') AS DOUBLE)").alias("lat")
).withColumn("geometry",
    expr("ST_Point(lon, lat)")
)

# Spatial operations on stream
result_df = parsed_df.join(
    zones_df,
    expr("ST_Contains(zone_geom, geometry)"),
    "inner"
)

# Aggregate by time window and zone
windowed_df = result_df.groupBy(
    window(expr("CAST(timestamp AS TIMESTAMP)"), "5 minutes"),
    "zone_id"
).count()

# Write stream to output
query = windowed_df.writeStream \
    .format("console") \
    .outputMode("update") \
    .start()

query.awaitTermination()
```

### Real-time Geofencing

```python
# Monitor events entering geofences
geofence_query = parsed_df.join(
    geofences_df,
    expr("ST_Contains(geofence_geom, geometry)"),
    "inner"
).select(
    "event_id",
    "timestamp",
    "geofence_id",
    "geofence_name"
)

# Alert on geofence entry
alert_query = geofence_query.writeStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("topic", "geofence_alerts") \
    .option("checkpointLocation", "/tmp/checkpoint") \
    .start()
```

---

## Hands-on Exercise 6: NYC Taxi Hotspot Analysis

### Objective
Analyze NYC taxi pickup patterns, create heatmaps, and identify hotspots.

### Code Solution

```python
from sedona.spark import *
from pyspark.sql.functions import expr, count, hour, floor
from pyspark.ml.clustering import KMeans
from pyspark.ml.feature import VectorAssembler

# Initialize
config = SedonaContext.builder().appName('module6').getOrCreate()
sedona = SedonaContext.create(config)

# 1. Load taxi trip data
trips = sedona.read.csv("data/nyc_taxi_trips.csv", header=True)
trips = trips.withColumn("pickup_point",
    expr("ST_Point(CAST(pickup_lon AS Decimal(24,20)), CAST(pickup_lat AS Decimal(24,20)))"))
trips = trips.withColumn("pickup_hour", hour("pickup_datetime"))

# 2. Create grid-based heatmap
print("=== Pickup Heatmap (Grid-Based) ===")
grid_size = 0.005

heatmap = sedona.sql(f"""
    SELECT
        FLOOR(ST_X(pickup_point) / {grid_size}) * {grid_size} as grid_x,
        FLOOR(ST_Y(pickup_point) / {grid_size}) * {grid_size} as grid_y,
        COUNT(*) as pickup_count
    FROM trips
    GROUP BY grid_x, grid_y
    HAVING pickup_count > 10
    ORDER BY pickup_count DESC
""")

heatmap.show(20)

# 3. Identify top hotspots
print("\n=== Top 10 Hotspots ===")
hotspots = heatmap.limit(10)
hotspots = hotspots.withColumn("hotspot_geom",
    expr(f"ST_MakeEnvelope(grid_x, grid_y, grid_x + {grid_size}, grid_y + {grid_size})"))

hotspots.select("grid_x", "grid_y", "pickup_count").show()

# 4. Temporal analysis (hourly patterns)
print("\n=== Pickup Patterns by Hour ===")
hourly = trips.groupBy("pickup_hour") \
    .agg(count("*").alias("trip_count")) \
    .orderBy("pickup_hour")

hourly.show(24)

# 5. Spatial clustering (K-means)
print("\n=== Spatial Clustering (K-Means) ===")

# Prepare features
trips_coords = trips.withColumn("x", expr("ST_X(pickup_point)")) \
                     .withColumn("y", expr("ST_Y(pickup_point)"))

assembler = VectorAssembler(inputCols=["x", "y"], outputCol="features")
trips_features = assembler.transform(trips_coords)

# Perform clustering
kmeans = KMeans(k=15, seed=42, maxIter=20)
model = kmeans.fit(trips_features)

# Get cluster centers
centers = model.clusterCenters()
print(f"\nCluster Centers:")
for i, center in enumerate(centers):
    print(f"Cluster {i}: lon={center[0]:.4f}, lat={center[1]:.4f}")

# 6. Join with points of interest
pois = sedona.read.csv("data/nyc_pois.csv", header=True)
pois = pois.withColumn("location",
    expr("ST_Point(CAST(lon AS Decimal(24,20)), CAST(lat AS Decimal(24,20)))"))

pois.createOrReplaceTempView("pois")
trips.createOrReplaceTempView("trips")

print("\n=== Trips Near Major POIs ===")
poi_trips = sedona.sql("""
    SELECT
        p.name,
        p.category,
        COUNT(*) as nearby_pickups
    FROM pois p
    JOIN trips t ON ST_DWithin(p.location, t.pickup_point, 0.002)
    GROUP BY p.name, p.category
    ORDER BY nearby_pickups DESC
    LIMIT 20
""")

poi_trips.show()

# 7. Export heatmap for visualization
print("\n=== Exporting Heatmap ===")

heatmap_export = heatmap.withColumn("cell_geom",
    expr(f"ST_MakeEnvelope(grid_x, grid_y, grid_x + {grid_size}, grid_y + {grid_size})"))

heatmap_export = heatmap_export.selectExpr(
    "grid_x",
    "grid_y",
    "pickup_count",
    "ST_AsGeoJSON(cell_geom) as geometry"
)

heatmap_export.coalesce(1).write.mode("overwrite") \
    .json("output/nyc_taxi_heatmap.geojson")

print("Heatmap exported to output/nyc_taxi_heatmap.geojson")

# 8. Statistical summary
print("\n=== Statistical Summary ===")
stats = sedona.sql("""
    SELECT
        COUNT(*) as total_trips,
        COUNT(DISTINCT DATE(pickup_datetime)) as days,
        COUNT(*) / COUNT(DISTINCT DATE(pickup_datetime)) as avg_trips_per_day
    FROM trips
""")

stats.show()
```

---

## Key Takeaways

- Spatial aggregations combine geometric operations with GROUP BY
- Grid-based heatmaps effectively visualize point density
- K-means and DBSCAN can cluster spatial data
- Sedona supports basic raster operations
- Export to GeoJSON for web visualization tools
- Structured Streaming enables real-time geospatial analytics
- Combine temporal and spatial analysis for deeper insights

---

## Visualization Tools Integration

| Tool | Format | Use Case |
|------|--------|----------|
| Kepler.gl | GeoJSON | Interactive exploration |
| Deck.gl | JSON | 3D visualizations |
| Folium | GeoJSON | Python notebooks |
| QGIS | Shapefile, GeoParquet | Desktop GIS |
| Tableau | CSV with WKT | Business analytics |
| D3.js | GeoJSON, TopoJSON | Custom web viz |

---

## Quiz Questions

1. What SQL function aggregates geometries into a single union?
2. How do you create a grid-based heatmap?
3. What's the difference between K-means and DBSCAN clustering?
4. What format should you use to export data for Kepler.gl?
5. How do you perform spatial operations on streaming data?
6. What is zonal statistics in raster processing?

---

**Previous:** [Module 5: Spatial Indexing and Partitioning](../module5/README.md) | **Next:** [Module 7: Real-world Case Studies](../module7/README.md)
