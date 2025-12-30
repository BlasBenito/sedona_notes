## Comparative Overview of Spatial Engines

| System          | Scope / Focus                             | Geometry Engine | Scalability / Execution Model |
|-----------------|------------------------------------------|----------------|-------------------------------|
| DuckDB (spatial)| Single-node analytical SQL + spatial ops | GEOS (C++)     | Single-node, multi-core vertical parallelism |
| PostGIS         | ACID-compliant relational + spatial SQL  | GEOS (C++)     | Single-node, vertical scaling, transactional |
| SedonaDB        | Single-node spatial-first SQL engine      | JTS (Java)     | Single-node, multi-core, spatial-aware |
| Apache Sedona   | Distributed large-scale spatial analytics| JTS (Java)     | Cluster-wide distributed execution, partition-aware |

---

## Detailed System Descriptions

### 1. DuckDB (spatial)
- **Scope:** Fast, single-node analytical database capable of handling both standard SQL and spatial queries. Ideal for prototyping and moderate-scale spatial analytics.
- **Internal Library:** Uses **GEOS (C++)** for geometry operations and **PROJ** for CRS transformations.
- **Scalability:** Single-node execution with multi-core parallelism. Scales vertically with CPU/RAM but cannot handle very large datasets beyond the capacity of one machine.
- **Use Case:** Exploratory spatial analysis, feature engineering, moderate spatial joins, and aggregations.

### 2. PostGIS
- **Scope:** ACID-compliant spatial extension to PostgreSQL. Supports full relational operations combined with spatial data types and SQL functions.
- **Internal Library:** Uses **GEOS (C++)** for all geometry operations. Integrates with **PROJ** and **GDAL/OGR** for CRS and I/O.
- **Scalability:** Single-node, vertical scaling. Highly robust and reliable for moderate to large datasets that fit on one machine.
- **Use Case:** Transactional spatial queries, mixed relational + spatial analytics, regional to national-scale datasets.

### 3. SedonaDB
- **Scope:** Single-node spatial-first SQL engine. Conceptually a bridge to distributed Sedona, designed to test and prototype distributed spatial algorithms on one machine.
- **Internal Library:** Uses **JTS (Java)** for geometry operations. Supports spatial indexing (R-tree, Quadtree) and spatial joins.
- **Scalability:** Single-node, multi-core, spatially aware. Scales better than generic RDBMS for spatial joins but limited to one machine.
- **Use Case:** Prototyping spatial algorithms, learning distributed Sedona concepts, regional-scale spatial analytics with complex joins.

### 4. Apache Sedona
- **Scope:** Distributed engine for large-scale spatial analytics. Integrates with Spark/Flink, supports massive datasets, and distributed computation of spatial joins, distance calculations, and neighborhood analyses.
- **Internal Library:** Uses **JTS (Java)** for geometry operations. Provides distributed spatial indexing, partitioning, and spatial query execution.
- **Scalability:** Horizontal scaling across a cluster. Designed to process hundreds of millions to billions of geometries efficiently.
- **Use Case:** Large-scale spatial autocorrelation, kNN graphs, distributed spatial joins, global-scale spatial analytics, iterative spatial computations.

---

## Key Takeaways
- **DuckDB & PostGIS** share GEOS (C++) for geometry, while **SedonaDB & Apache Sedona** use JTS (Java). GEOS is a port of JTS, ensuring consistent geometry semantics.
- **PostGIS** excels at robust, transactional, single-node spatial queries. **DuckDB** is fast for analytical workloads on one machine.
- **SedonaDB** allows single-node experimentation with distributed Sedona concepts. **Apache Sedona** handles truly large-scale distributed spatial computation.
- For very large datasets or high-complexity spatial analytics (e.g., local Moran, LISA, large kNN graphs), **Apache Sedona** is the practical choice, while PostGIS and DuckDB are suitable for smaller-scale or prototyping workloads.

