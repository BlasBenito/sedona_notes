# R Guide for Distributed Sedona

## Overview
This guide shows how to use Apache Sedona from R using the `apache.sedona` package with SparkR or sparklyr.

---

## Installation

### Option 1: Using SparkR (Recommended)

```r
# Install SparkR (comes with Spark)
install.packages("SparkR")

# Start Spark with Sedona
library(SparkR)

sparkR.session(
  appName = "SedonaR",
  sparkPackages = c(
    "org.apache.sedona:sedona-spark-shaded-3.0_2.12:1.5.1",
    "org.datasyslab:geotools-wrapper:1.5.1-28.2"
  )
)

# Register Sedona functions
sql("SELECT 1")  # Initialize SQL
```

### Option 2: Using sparklyr

```r
install.packages("sparklyr")
library(sparklyr)

# Configure Spark with Sedona
config <- spark_config()
config$spark.jars.packages <- "org.apache.sedona:sedona-spark-shaded-3.0_2.12:1.5.1,org.datasyslab:geotools-wrapper:1.5.1-28.2"

# Connect
sc <- spark_connect(master = "local", config = config)
```

---

## Quick Start Example

### Using SparkR

```r
library(SparkR)

# Initialize with Sedona
sparkR.session(
  appName = "SedonaExample",
  sparkPackages = "org.apache.sedona:sedona-spark-shaded-3.0_2.12:1.5.1,org.datasyslab:geotools-wrapper:1.5.1-28.2"
)

# Create sample data
df <- createDataFrame(data.frame(
  id = 1:3,
  name = c("Point A", "Point B", "Point C"),
  lon = c(-74.0, -73.9, -74.1),
  lat = c(40.7, 40.8, 40.6)
))

# Register as temp view
createOrReplaceTempView(df, "points")

# Create geometries with Sedona SQL
result <- sql("
  SELECT
    id,
    name,
    ST_Point(lon, lat) as geometry,
    ST_AsText(ST_Point(lon, lat)) as wkt
  FROM points
")

# Show results
showDF(result)

# Collect to R
points_r <- collect(result)
print(points_r)
```

---

## Common Operations

### Loading Data

```r
# Load CSV with coordinates
df <- read.df("data/locations.csv", "csv", header = "true")

# Create geometries
createOrReplaceTempView(df, "locations")
spatial_df <- sql("
  SELECT
    *,
    ST_Point(CAST(lon AS DOUBLE), CAST(lat AS DOUBLE)) as geom
  FROM locations
")
```

### Spatial Queries

```r
# Point-in-polygon query
createOrReplaceTempView(spatial_df, "points")
createOrReplaceTempView(polygons_df, "polygons")

result <- sql("
  SELECT p.id, p.name, poly.zone_name
  FROM points p
  JOIN polygons poly ON ST_Contains(poly.geometry, p.geom)
")

showDF(result)
```

### Distance Queries

```r
# Find points within 1km
result <- sql("
  SELECT
    id,
    name,
    ST_Distance(
      ST_Transform(ST_SetSRID(geom, 4326), 3857),
      ST_Transform(ST_SetSRID(ST_Point(-74.0, 40.7), 4326), 3857)
    ) as distance_m
  FROM points
  WHERE ST_DWithin(
    ST_Transform(ST_SetSRID(geom, 4326), 3857),
    ST_Transform(ST_SetSRID(ST_Point(-74.0, 40.7), 4326), 3857),
    1000.0
  )
  ORDER BY distance_m
")

showDF(result)
```

### Spatial Aggregations

```r
# Count points per zone
result <- sql("
  SELECT
    z.zone_id,
    z.zone_name,
    COUNT(p.id) as point_count
  FROM zones z
  LEFT JOIN points p ON ST_Contains(z.geometry, p.geom)
  GROUP BY z.zone_id, z.zone_name
  ORDER BY point_count DESC
")

showDF(result)
```

---

## Using sparklyr

```r
library(sparklyr)
library(dplyr)

# Connect
config <- spark_config()
config$spark.jars.packages <- "org.apache.sedona:sedona-spark-shaded-3.0_2.12:1.5.1,org.datasyslab:geotools-wrapper:1.5.1-28.2"
sc <- spark_connect(master = "local", config = config)

# Load data
locations <- spark_read_csv(sc, "locations", "data/locations.csv")

# Register temp view
sdf_register(locations, "locations_tbl")

# Use Sedona SQL via DBI
library(DBI)
result <- dbGetQuery(sc, "
  SELECT
    *,
    ST_Point(CAST(lon AS DOUBLE), CAST(lat AS DOUBLE)) as geom
  FROM locations_tbl
")

# Or use dplyr
result <- locations %>%
  mutate(geom = sql("ST_Point(CAST(lon AS DOUBLE), CAST(lat AS DOUBLE))"))
```

---

## Working with sf Package

### Convert Sedona Results to sf

```r
library(sf)

# Query with Sedona
result <- sql("
  SELECT
    id,
    name,
    ST_AsText(geometry) as wkt
  FROM spatial_data
")

# Collect to R
result_r <- collect(result)

# Convert to sf object
result_sf <- st_as_sf(result_r, wkt = "wkt")

# Now use sf functions
plot(result_sf)
st_crs(result_sf) <- 4326
```

### Load sf Data into Sedona

```r
library(sf)

# Read with sf
data_sf <- st_read("data/boundaries.shp")

# Convert to SparkR DataFrame
data_r <- data_sf %>%
  mutate(wkt = st_as_text(geometry)) %>%
  st_drop_geometry()

data_spark <- createDataFrame(data_r)

# Parse WKT in Sedona
createOrReplaceTempView(data_spark, "temp_data")
result <- sql("
  SELECT
    *,
    ST_GeomFromWKT(wkt) as geometry
  FROM temp_data
")
```

---

## Complete Example: NYC Taxi Analysis

```r
library(SparkR)

# Initialize Spark with Sedona
sparkR.session(
  appName = "NYCTaxiAnalysis",
  master = "local[*]",
  sparkPackages = "org.apache.sedona:sedona-spark-shaded-3.0_2.12:1.5.1,org.datasyslab:geotools-wrapper:1.5.1-28.2",
  sparkConfig = list(
    spark.executor.memory = "4g",
    spark.driver.memory = "2g"
  )
)

# Load taxi trip data
trips <- read.df("data/nyc_taxi_trips.csv", "csv", header = "true")

# Create geometries
createOrReplaceTempView(trips, "trips_raw")
trips_spatial <- sql("
  SELECT
    *,
    ST_Point(
      CAST(pickup_longitude AS DOUBLE),
      CAST(pickup_latitude AS DOUBLE)
    ) as pickup_geom
  FROM trips_raw
  WHERE pickup_longitude IS NOT NULL
    AND pickup_latitude IS NOT NULL
")

# Load borough boundaries
boroughs <- read.df("data/nyc_boroughs.geojson", "json")
createOrReplaceTempView(boroughs, "boroughs_raw")
boroughs_spatial <- sql("
  SELECT
    properties.borough_name as name,
    ST_GeomFromGeoJSON(geometry) as geom
  FROM boroughs_raw
")

# Join trips with boroughs
createOrReplaceTempView(trips_spatial, "trips")
createOrReplaceTempView(boroughs_spatial, "boroughs")

result <- sql("
  SELECT
    b.name as borough,
    COUNT(*) as trip_count,
    AVG(CAST(t.fare_amount AS DOUBLE)) as avg_fare
  FROM trips t
  JOIN boroughs b ON ST_Contains(b.geom, t.pickup_geom)
  GROUP BY b.name
  ORDER BY trip_count DESC
")

# Show results
showDF(result)

# Collect to R for plotting
result_r <- collect(result)

# Plot with ggplot2
library(ggplot2)
ggplot(result_r, aes(x = reorder(borough, -trip_count), y = trip_count)) +
  geom_bar(stat = "identity", fill = "steelblue") +
  theme_minimal() +
  labs(
    title = "NYC Taxi Trips by Borough",
    x = "Borough",
    y = "Number of Trips"
  ) +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```

---

## Performance Tips for R

### 1. Avoid Collecting Large DataFrames

```r
# Bad: Collect entire dataset
all_data <- collect(large_df)  # May crash

# Good: Aggregate first, then collect
summary <- sql("
  SELECT zone, COUNT(*) as count, AVG(value) as avg_value
  FROM large_table
  GROUP BY zone
")
summary_r <- collect(summary)
```

### 2. Use Broadcast for Small Tables

```r
# Cache small reference data
createOrReplaceTempView(zones, "zones")
sql("CACHE TABLE zones")

# Now joins will be faster
result <- sql("
  SELECT p.*, z.zone_name
  FROM large_points p
  JOIN zones z ON ST_Contains(z.geom, p.geom)
")
```

### 3. Partition Data

```r
# Repartition for better parallelism
repartitioned <- repartition(df, numPartitions = 200)
createOrReplaceTempView(repartitioned, "partitioned_data")
```

---

## Exporting Results

### To CSV

```r
# Write results
write.df(
  result,
  path = "output/results.csv",
  source = "csv",
  mode = "overwrite",
  header = "true"
)
```

### To GeoParquet

```r
# Write spatial data
write.df(
  spatial_df,
  path = "output/spatial_data.parquet",
  source = "parquet",
  mode = "overwrite"
)
```

### To sf for Local Processing

```r
# Collect and convert to sf
result_wkt <- sql("
  SELECT
    id,
    name,
    ST_AsText(geometry) as wkt
  FROM spatial_data
  LIMIT 10000
")

result_r <- collect(result_wkt)
result_sf <- st_as_sf(result_r, wkt = "wkt", crs = 4326)

# Save as shapefile
st_write(result_sf, "output/results.shp")
```

---

## Comparison: SparkR vs sparklyr

| Feature | SparkR | sparklyr |
|---------|--------|----------|
| **Maintained by** | Apache Spark | RStudio |
| **dplyr support** | No | Yes |
| **SQL support** | Excellent | Good |
| **Performance** | Slightly faster | Good |
| **R integration** | Basic | Better |
| **Learning curve** | Lower | Higher |
| **Best for** | SQL-focused work | dplyr users |

---

## Troubleshooting

### Issue: Sedona functions not found

```r
# Solution: Ensure packages are loaded
sparkR.session(
  sparkPackages = c(
    "org.apache.sedona:sedona-spark-shaded-3.0_2.12:1.5.1",
    "org.datasyslab:geotools-wrapper:1.5.1-28.2"
  )
)

# Test
sql("SELECT ST_Point(0, 0)")
```

### Issue: Out of memory

```r
# Increase memory
sparkR.session(
  sparkConfig = list(
    spark.executor.memory = "8g",
    spark.driver.memory = "4g"
  )
)
```

### Issue: Slow queries

```r
# Check execution plan
df <- sql("EXPLAIN SELECT ... FROM ...")
showDF(df, truncate = FALSE)

# Create spatial index (via SQL hint)
sql("
  SELECT /*+ BROADCAST(zones) */ *
  FROM points p
  JOIN zones z ON ST_Contains(z.geom, p.geom)
")
```

---

## Additional Resources

- [SparkR Documentation](https://spark.apache.org/docs/latest/sparkr.html)
- [sparklyr Documentation](https://spark.rstudio.com/)
- [Apache Sedona SQL Functions](https://sedona.apache.org/latest-snapshot/api/sql/Overview/)
- [sf Package](https://r-spatial.github.io/sf/)

---

**Back to:** [Part 2 Overview](README.md) | [Python Quick Reference](QUICK_REFERENCE.md)
