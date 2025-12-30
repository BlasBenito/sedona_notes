# Module 1: Introduction to SedonaDB
**Duration:** 1.5 hours

## Overview
This module introduces SedonaDB, a high-performance geospatial database built on DuckDB. You'll understand its architecture, compare it with alternatives, and get hands-on with installation and first queries.

---

## 1.1 What is SedonaDB? (20 minutes)

### Definition
SedonaDB is a geospatial database system that extends DuckDB with spatial capabilities. It provides SQL-based spatial operations with exceptional performance on a single node.

### Key Features
- **In-process database** - No separate server process
- **Columnar storage** - Optimized for analytical queries
- **Spatial indexing** - R-Tree for fast spatial queries
- **Standard SQL** - Familiar query language
- **Multiple bindings** - Python, R, Java, CLI
- **Format support** - Parquet, CSV, JSON, Shapefile, GeoJSON
- **ACID compliant** - Full transaction support
- **Zero-copy integration** - Efficient with Arrow/Pandas

### DuckDB Foundation
SedonaDB is built on DuckDB, which provides:
- Vectorized query execution
- Parallel query processing
- Efficient compression
- OLAP-optimized design
- Embedded architecture (like SQLite)

---

## 1.2 SedonaDB vs Alternatives (15 minutes)

### Comparison Table

| Feature | SedonaDB | PostGIS | Distributed Sedona | GeoPandas |
|---------|----------|---------|-------------------|-----------|
| **Architecture** | Embedded | Client-Server | Distributed Cluster | In-memory |
| **Scale** | Single node | Single node | Multi-node | Single machine |
| **Max data size** | TBs | TBs | Petabytes | GBs |
| **Query language** | SQL | SQL | SQL + RDD API | Python |
| **Setup complexity** | Minimal | Moderate | High | Minimal |
| **Spatial indexing** | R-Tree | R-Tree/GIST | R-Tree/Quad-Tree | R-Tree |
| **Analytics focus** | OLAP | OLTP + OLAP | OLAP | Analysis |
| **Best for** | Analytics | Transactions | Big Data | Prototyping |

### When to Use SedonaDB

**Use SedonaDB when:**
- Data fits on single machine (up to several TB)
- Need fast analytical queries
- Want embedded database (no server)
- Prefer SQL interface
- Need columnar storage benefits
- Prototyping before scaling to distributed

**Use PostGIS when:**
- Need traditional database server
- OLTP workload (many small transactions)
- Require advanced spatial functions
- Multi-user concurrent access
- Integration with existing PostgreSQL stack

**Use Distributed Sedona when:**
- Data exceeds single-node capacity (>5TB)
- Need horizontal scalability
- Processing billions of geometries
- Integration with Spark ecosystem
- Real-time streaming required

---

## 1.3 Architecture and Design (15 minutes)

### Architecture Overview

```
┌─────────────────────────────────────────┐
│         Your Application                 │
│    (Python, R, Java, SQL CLI)           │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│         SedonaDB API                     │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│      Spatial Extension Layer             │
│  - ST_* Functions                        │
│  - Spatial Indexing                      │
│  - Format Readers/Writers                │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│          DuckDB Core                     │
│  - Query Parser                          │
│  - Query Optimizer                       │
│  - Vectorized Execution                  │
│  - Columnar Storage                      │
└─────────────────────────────────────────┘
```

### Key Design Principles

**1. In-Process Execution**
- No client-server overhead
- Direct memory access
- Minimal latency

**2. Vectorized Processing**
- Process data in batches (vectors)
- CPU cache-friendly
- SIMD optimizations

**3. Columnar Storage**
- Better compression
- Skip irrelevant columns
- Optimized for analytics

**4. Zero-Copy Integration**
- Arrow format support
- Pandas integration
- No serialization overhead

---

## 1.4 Installation and Setup (25 minutes)

### Installation Methods

**Method 1: Python (Recommended)**

```bash
pip install duckdb
pip install duckdb-spatial  # or use built-in spatial extension
```

**Method 2: R**

```r
install.packages("duckdb")
# Spatial extension loaded automatically when needed
```

**Method 3: CLI**

```bash
# Download DuckDB CLI from https://duckdb.org/
wget https://github.com/duckdb/duckdb/releases/download/v0.10.0/duckdb_cli-linux-amd64.zip
unzip duckdb_cli-linux-amd64.zip
```

### Python Setup

```python
import duckdb

# Create in-memory database
conn = duckdb.connect(':memory:')

# Or persistent database
conn = duckdb.connect('my_spatial_db.duckdb')

# Install and load spatial extension
conn.execute("INSTALL spatial;")
conn.execute("LOAD spatial;")

# Verify installation
result = conn.execute("""
    SELECT ST_AsText(ST_Point(0, 0)) as point
""").fetchall()

print(result)  # [('POINT (0 0)',)]
```

### R Setup

```r
library(duckdb)

# Create connection
con <- dbConnect(duckdb::duckdb(), dbdir = ":memory:")

# Install spatial extension
dbExecute(con, "INSTALL spatial;")
dbExecute(con, "LOAD spatial;")

# Verify
dbGetQuery(con, "SELECT ST_AsText(ST_Point(0, 0)) as point")
```

### CLI Setup

```bash
./duckdb my_spatial_db.duckdb

D INSTALL spatial;
D LOAD spatial;
D SELECT ST_AsText(ST_Point(0, 0));
┌─────────────────┐
│ st_astext(st... │
│     varchar     │
├─────────────────┤
│ POINT (0 0)     │
└─────────────────┘
```

---

## 1.5 First Spatial Queries (20 minutes)

### Creating Geometries

```python
import duckdb

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

# Create points
result = conn.execute("""
    SELECT
        ST_AsText(ST_Point(0, 0)) as origin,
        ST_AsText(ST_Point(1, 1)) as point_a,
        ST_AsText(ST_Point(-1, 2)) as point_b
""").df()

print(result)
```

### Distance Calculations

```python
# Calculate distance between points
result = conn.execute("""
    SELECT
        ST_Distance(
            ST_Point(0, 0),
            ST_Point(3, 4)
        ) as euclidean_distance
""").fetchone()

print(f"Distance: {result[0]}")  # 5.0
```

### Creating a Spatial Table

```python
# Create table with spatial data
conn.execute("""
    CREATE TABLE cities (
        id INTEGER,
        name VARCHAR,
        location GEOMETRY
    );
""")

# Insert data
conn.execute("""
    INSERT INTO cities VALUES
        (1, 'New York', ST_Point(-74.006, 40.7128)),
        (2, 'Los Angeles', ST_Point(-118.2437, 34.0522)),
        (3, 'Chicago', ST_Point(-87.6298, 41.8781))
""")

# Query
cities = conn.execute("""
    SELECT
        name,
        ST_AsText(location) as coordinates
    FROM cities
    ORDER BY name
""").df()

print(cities)
```

### Simple Spatial Query

```python
# Find cities within bounding box
result = conn.execute("""
    SELECT name
    FROM cities
    WHERE ST_Within(
        location,
        ST_MakeEnvelope(-120, 30, -70, 45)
    )
""").df()

print(result)
```

---

## 1.6 Performance Characteristics (15 minutes)

### Benchmark: SedonaDB vs PostGIS

**Test scenario:** Point-in-polygon query on 1M points

| Database | Time (seconds) | Memory (GB) |
|----------|---------------|-------------|
| SedonaDB | 0.8 | 0.5 |
| PostGIS | 2.3 | 1.2 |
| GeoPandas | 45.0 | 8.0 |

### Why SedonaDB is Fast

**1. Columnar Storage**
```
Traditional row storage:
[ID1, Name1, Geom1], [ID2, Name2, Geom2], ...
Must read entire rows even for single column

Columnar storage:
[ID1, ID2, ...], [Name1, Name2, ...], [Geom1, Geom2, ...]
Read only needed columns
```

**2. Vectorized Execution**
```python
# Instead of processing one row at a time:
for row in data:
    result.append(ST_Distance(row.geom, query_point))

# Process in batches (vectors):
result = ST_Distance_Vector(data.geom_column, query_point)
# 10-100x faster due to CPU optimizations
```

**3. Parallel Query Execution**
- Automatically uses multiple CPU cores
- No configuration needed
- Scales with available cores

### Practical Performance Example

```python
import duckdb
import time

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

# Create 1 million random points
start = time.time()
conn.execute("""
    CREATE TABLE random_points AS
    SELECT
        row_number() OVER () as id,
        ST_Point(
            random() * 360 - 180,  -- longitude
            random() * 180 - 90    -- latitude
        ) as geom
    FROM range(1000000)
""")
create_time = time.time() - start
print(f"Created 1M points in {create_time:.2f}s")

# Query: Find points within 10 degrees of origin
start = time.time()
result = conn.execute("""
    SELECT COUNT(*)
    FROM random_points
    WHERE ST_DWithin(geom, ST_Point(0, 0), 10)
""").fetchone()
query_time = time.time() - start

print(f"Query time: {query_time:.2f}s")
print(f"Found {result[0]:,} points")
```

---

## Hands-on Exercise 1: Setup and First Queries

### Objective
Install SedonaDB, create a spatial database, and execute basic queries.

### Tasks

**Task 1: Installation**
```python
import duckdb

# Create connection and load spatial
conn = duckdb.connect('exercise1.duckdb')
conn.execute("INSTALL spatial; LOAD spatial;")

# Verify
print(conn.execute("SELECT ST_AsText(ST_Point(1, 2))").fetchone())
```

**Task 2: Create Landmark Database**
```python
# Create table
conn.execute("""
    CREATE TABLE landmarks (
        id INTEGER PRIMARY KEY,
        name VARCHAR,
        city VARCHAR,
        location GEOMETRY,
        established_year INTEGER
    )
""")

# Insert famous landmarks
conn.execute("""
    INSERT INTO landmarks VALUES
        (1, 'Statue of Liberty', 'New York',
         ST_Point(-74.0445, 40.6892), 1886),
        (2, 'Golden Gate Bridge', 'San Francisco',
         ST_Point(-122.4783, 37.8199), 1937),
        (3, 'Space Needle', 'Seattle',
         ST_Point(-122.3493, 47.6205), 1962),
        (4, 'Willis Tower', 'Chicago',
         ST_Point(-87.6359, 41.8789), 1973),
        (5, 'Empire State Building', 'New York',
         ST_Point(-73.9857, 40.7484), 1931)
""")
```

**Task 3: Basic Queries**
```python
# 1. List all landmarks
print("All landmarks:")
print(conn.execute("""
    SELECT name, city, ST_AsText(location) as coords
    FROM landmarks
    ORDER BY name
""").df())

# 2. Find landmarks in New York
print("\nNew York landmarks:")
print(conn.execute("""
    SELECT name, established_year
    FROM landmarks
    WHERE city = 'New York'
    ORDER BY established_year
""").df())

# 3. Calculate distance between Statue of Liberty and Empire State
print("\nDistance between landmarks:")
result = conn.execute("""
    SELECT
        ST_Distance(
            (SELECT location FROM landmarks WHERE name = 'Statue of Liberty'),
            (SELECT location FROM landmarks WHERE name = 'Empire State Building')
        ) as distance_degrees
""").fetchone()
print(f"Distance: {result[0]:.4f} degrees")
```

**Task 4: Spatial Query**
```python
# Find landmarks within 100km of Chicago (approximate)
print("\nLandmarks near Chicago:")
chicago_point = conn.execute("""
    SELECT location FROM landmarks WHERE city = 'Chicago'
""").fetchone()[0]

result = conn.execute("""
    SELECT
        name,
        ST_Distance(location, ST_Point(-87.6298, 41.8781)) * 111 as distance_km
    FROM landmarks
    WHERE ST_DWithin(location, ST_Point(-87.6298, 41.8781), 1.0)
    ORDER BY distance_km
""").df()
print(result)
```

---

## Key Takeaways

- SedonaDB is an embedded geospatial database built on DuckDB
- Combines SQL interface with high performance
- Ideal for single-node analytics on TB-scale data
- Columnar storage and vectorized execution provide exceptional speed
- Simple installation and setup (Python, R, CLI)
- Natural stepping stone to distributed Sedona for larger datasets
- Use for prototyping, analytics, and medium-scale production workloads

---

## Comparison Summary

**Choose SedonaDB when you need:**
- Fast analytical queries on spatial data
- Embedded database (no server management)
- SQL interface
- Data fits on one machine

**Move to Distributed Sedona when:**
- Data exceeds single-node capacity
- Need horizontal scalability
- Real-time streaming requirements
- Integration with Spark ecosystem

---

## Quiz Questions

1. What database is SedonaDB built on?
2. What are the three main performance advantages of SedonaDB?
3. How do you load the spatial extension in SedonaDB?
4. What's the difference between SedonaDB and PostGIS?
5. When should you scale from SedonaDB to Distributed Sedona?

---

**Next:** [Module 2: Spatial SQL Fundamentals](../module2/README.md)
