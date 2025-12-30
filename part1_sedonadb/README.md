# Part 1: SedonaDB - Geospatial Database Fundamentals
## 10-Hour Course

Learn high-performance single-node geospatial database operations with SedonaDB before scaling to distributed systems.

---

## Quick Start

```python
import duckdb

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

# Your first spatial query
result = conn.execute("""
    SELECT ST_AsText(ST_Point(-74.006, 40.7128)) as nyc
""").fetchone()
print(result)
```

---

## Course Modules

1. **[Introduction to SedonaDB](modules/module1/README.md)** (1.5h)
2. **[Spatial SQL Fundamentals](modules/module2/README.md)** (1.5h)
3. **[Advanced Spatial Queries](modules/module3/README.md)** (2h)
4. **[Data Integration and ETL](modules/module4/README.md)** (1.5h)
5. **[Performance Optimization](modules/module5/README.md)** (1.5h)
6. **[Application Development](modules/module6/README.md)** (1.5h)
7. **[Production Deployment](modules/module7/README.md)** (1.5h)

**[Exercises and Projects](exercises/README.md)**

---

## What You'll Learn

- Build spatial databases with SedonaDB
- Master spatial SQL operations
- Optimize query performance with indexing
- Build production applications (Python/R)
- Deploy spatial web services
- Know when to scale to distributed systems

---

## Next Steps

After completing Part 1, continue to:
**[Part 2: Distributed Sedona](../part2_distributed/README.md)** - Scale to billions of geometries

---

**Back to:** [Main Course](../README.md)
