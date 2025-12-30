# Module 5: Performance Optimization
**Duration:** 1.5 hours

## 5.1 Spatial Indexing (25 min)

```sql
-- Create R-Tree spatial index
CREATE INDEX idx_spatial ON table_name USING RTREE (geom);

-- Check if index is used
EXPLAIN SELECT * FROM table_name WHERE ST_Contains(geom, ST_Point(0, 0));

-- Index effectiveness
PRAGMA show_tables_expanded;
```

## 5.2 Query Optimization (30 min)

```sql
-- EXPLAIN to analyze query plan
EXPLAIN ANALYZE
SELECT *
FROM large_table
WHERE ST_DWithin(geom, ST_Point(0, 0), 0.1);

-- Filter before spatial operations
SELECT *
FROM (SELECT * FROM large_table WHERE category = 'restaurant')
WHERE ST_DWithin(geom, ST_Point(0, 0), 0.1);

-- Use bounding box for initial filter
SELECT *
FROM large_table
WHERE ST_Within(geom, ST_MakeEnvelope(-1, -1, 1, 1))
    AND ST_DWithin(geom, ST_Point(0, 0), 0.1);

-- Simplify geometries
SELECT ST_Simplify(geom, 0.001) FROM complex_polygons;
```

## 5.3 Memory and Parallelism (20 min)

```python
import duckdb

conn = duckdb.connect()

# Set memory limit
conn.execute("SET memory_limit='4GB'")

# Set thread count
conn.execute("SET threads=8")

# Enable optimizer
conn.execute("SET enable_optimizer=true")

# Batch inserts for performance
conn.execute("BEGIN TRANSACTION")
for batch in data_batches:
    conn.executemany("INSERT INTO table VALUES (?, ?, ?)", batch)
conn.execute("COMMIT")
```

## 5.4 Materialized Views (15 min)

```sql
-- Create materialized view for expensive query
CREATE TABLE district_stats AS
SELECT
    district_id,
    COUNT(*) as total_properties,
    AVG(price) as avg_price,
    ST_Union_Agg(geom) as district_boundary
FROM properties
GROUP BY district_id;

-- Refresh when data changes
DROP TABLE district_stats;
CREATE TABLE district_stats AS ...;
```

## 5.5 When to Scale to Distributed (10 min)

**Move to Distributed Sedona when:**
- Data > 100GB on disk
- Queries take > 10 minutes despite optimization
- Need real-time processing of streams
- Horizontal scaling required
- Integration with Spark ecosystem needed

---

**Previous:** [Module 4](../module4/README.md) | **Next:** [Module 6](../module6/README.md)
