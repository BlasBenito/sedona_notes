# Apache Sedona: Complete Geospatial Data Processing Course
## 20 Hours - From Database to Distributed Systems

Master geospatial data processing from single-node databases to distributed cluster computing.

---

## Course Structure

This comprehensive course is divided into two parts:

### **[Part 1: SedonaDB](part1_sedonadb/README.md)** - 10 Hours
**Single-Node Geospatial Database**

Start with SedonaDB, a high-performance embedded spatial database. Learn SQL-based geospatial operations, indexing, and application development on a single machine.

**Best for:**
- Data up to several TB
- Fast analytical queries
- Embedded applications
- Learning fundamentals

**Topics:**
- Spatial SQL fundamentals
- Advanced spatial queries
- Data integration and ETL
- Performance optimization
- Application development (Python/R)
- Production deployment

---

### **[Part 2: Distributed Sedona](part2_distributed/README.md)** - 10 Hours
**Big Data Geospatial Processing**

Scale to billions of geometries with Apache Sedona on Spark. Process massive datasets across distributed clusters with spatial indexing and partitioning.

**Best for:**
- Data > 100GB
- Billions of geometries
- Real-time streaming
- Horizontal scaling

**Topics:**
- Distributed spatial RDDs
- Spatial queries and joins at scale
- Spatial indexing and partitioning
- Advanced analytics
- Real-time streaming
- Production case studies

---

## Learning Path

```
┌─────────────────────────────────────────┐
│  Part 1: SedonaDB (10 hours)            │
│  ├─ Module 1: Introduction              │
│  ├─ Module 2: Spatial SQL               │
│  ├─ Module 3: Advanced Queries          │
│  ├─ Module 4: Data Integration          │
│  ├─ Module 5: Performance               │
│  ├─ Module 6: Applications              │
│  └─ Module 7: Production                │
└──────────────┬──────────────────────────┘
               │
               │ When to scale?
               │ • Data > 100GB
               │ • Need distributed processing
               │
┌──────────────▼──────────────────────────┐
│  Part 2: Distributed Sedona (10 hours)  │
│  ├─ Module 1: Introduction              │
│  ├─ Module 2: Spatial Data              │
│  ├─ Module 3: Spatial RDDs              │
│  ├─ Module 4: Queries and Joins         │
│  ├─ Module 5: Indexing/Partitioning     │
│  ├─ Module 6: Advanced Analytics        │
│  └─ Module 7: Case Studies              │
└─────────────────────────────────────────┘
```

---

## Quick Start

### Part 1: SedonaDB

```python
import duckdb

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")

# Create spatial data
conn.execute("""
    CREATE TABLE locations AS
    SELECT
        'New York' as city,
        ST_Point(-74.006, 40.7128) as location
""")

# Query
result = conn.execute("""
    SELECT city, ST_AsText(location)
    FROM locations
""").fetchall()
```

### Part 2: Distributed Sedona

```python
from sedona.spark import *

config = SedonaContext.builder().appName('sedona').getOrCreate()
sedona = SedonaContext.create(config)

# Create spatial DataFrame
df = sedona.sql("""
    SELECT ST_Point(-74.006, 40.7128) as location
""")

df.show()
```

---

## Prerequisites

### Part 1 (SedonaDB)
- Basic SQL knowledge
- Python or R (for application modules)
- Understanding of geospatial concepts

### Part 2 (Distributed Sedona)
- **Complete Part 1** or equivalent experience
- Basic Apache Spark knowledge (helpful)
- Python or Scala programming

---

## Course Materials

```
sedona_notes/
├── README.md                    # This file
├── part1_sedonadb/              # Part 1: Database
│   ├── README.md
│   ├── COURSE_OUTLINE.md
│   ├── modules/
│   │   ├── module1/             # Introduction
│   │   ├── module2/             # Spatial SQL
│   │   ├── module3/             # Advanced Queries
│   │   ├── module4/             # Data Integration
│   │   ├── module5/             # Performance
│   │   ├── module6/             # Applications
│   │   └── module7/             # Production
│   └── exercises/
└── part2_distributed/           # Part 2: Distributed
    ├── README.md
    ├── COURSE_OUTLINE.md
    ├── QUICK_REFERENCE.md
    ├── modules/
    │   ├── module1/             # Introduction
    │   ├── module2/             # Spatial Data
    │   ├── module3/             # Spatial RDDs
    │   ├── module4/             # Queries & Joins
    │   ├── module5/             # Indexing
    │   ├── module6/             # Analytics
    │   └── module7/             # Case Studies
    └── exercises/
```

---

## Recommended Study Plans

### **Beginner Path** (12-14 hours)
- Complete all Part 1 modules
- Part 1 exercises
- Skip Part 2 until needed

### **Intermediate Path** (16-18 hours)
- Complete Part 1 modules
- Part 1 final project
- Part 2 modules 1-4
- Part 2 exercises

### **Advanced Path** (20+ hours)
- Complete all modules in both parts
- All exercises and projects
- Bonus challenges
- Deploy production applications

---

## What You'll Build

### Part 1 Projects
- Restaurant finder application
- Real estate search platform
- Urban planning dashboard

### Part 2 Projects
- NYC taxi analytics pipeline
- Urban mobility analysis system
- Real-time geofencing service

---

## Key Technologies

**Part 1:**
- DuckDB
- SedonaDB spatial extension
- Python (duckdb, pandas, fastapi)
- R (duckdb package)

**Part 2:**
- Apache Spark
- Apache Sedona
- PySedona / Scala / R (SparkR)
- Spark SQL & Streaming

---

## When to Use Each Part

### Use SedonaDB (Part 1) when:
- Data fits on one machine (< 5TB)
- Need embedded database
- Fast prototyping
- Simple deployment
- Cost-effective single-node processing

### Use Distributed Sedona (Part 2) when:
- Data > 100GB
- Processing billions of geometries
- Need horizontal scaling
- Real-time streaming required
- Integration with Spark ecosystem

---

## Learning Outcomes

After completing both parts, you will:

**Technical Skills:**
- [ ] Build spatial databases with SedonaDB
- [ ] Write complex spatial SQL queries
- [ ] Optimize single-node performance
- [ ] Develop spatial REST APIs
- [ ] Process TB-scale data on single node
- [ ] Scale to distributed Spark clusters
- [ ] Perform spatial joins on billions of geometries
- [ ] Build spatial indexes and partitioning strategies
- [ ] Create real-time spatial analytics pipelines
- [ ] Deploy production geospatial applications

**Strategic Skills:**
- [ ] Choose appropriate technology for scale
- [ ] Design efficient spatial schemas
- [ ] Optimize query performance
- [ ] Plan migration paths
- [ ] Make build vs buy decisions

---

## Getting Started

### Installation

**Part 1 (SedonaDB):**
```bash
pip install duckdb
```

**Part 2 (Distributed Sedona):**
```bash
pip install apache-sedona
```

### Begin Learning

1. **Start here:** [Part 1: SedonaDB](part1_sedonadb/README.md)
2. **Then progress to:** [Part 2: Distributed Sedona](part2_distributed/README.md)

---

## Additional Resources

### Documentation
- [DuckDB Documentation](https://duckdb.org/docs/)
- [DuckDB Spatial Extension](https://duckdb.org/docs/extensions/spatial.html)
- [Apache Sedona Documentation](https://sedona.apache.org/)
- [Apache Spark Documentation](https://spark.apache.org/docs/latest/)

### Community
- [DuckDB GitHub](https://github.com/duckdb/duckdb)
- [Apache Sedona GitHub](https://github.com/apache/sedona)
- [Sedona Community Slack](https://sedona-community.slack.com/)

### Sample Datasets
- [NYC Open Data](https://opendata.cityofnewyork.us/)
- [OpenStreetMap](https://www.openstreetmap.org/)
- [Natural Earth](https://www.naturalearthdata.com/)
- [US Census TIGER/Line](https://www.census.gov/geographies/mapping-files.html)

---

## Contributing

Found an error or want to improve the course?

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

---

## License

This course material is provided for educational purposes.
- Apache Sedona: Apache License 2.0
- DuckDB: MIT License

---

## Acknowledgments

This comprehensive course was created to provide a complete learning path from single-node geospatial databases to distributed big data processing. Special thanks to the DuckDB and Apache Sedona communities.

---

## Start Your Journey

Ready to master geospatial data processing?

**Begin with:** [Part 1: SedonaDB - Geospatial Database Fundamentals](part1_sedonadb/README.md)

---

**Happy Learning!** 🗺️ 🚀
