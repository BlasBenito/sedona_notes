# Module 1: Introduction to Apache Sedona
**Duration:** 1 hour

## Overview
This module introduces Apache Sedona, its architecture, capabilities, and position in the geospatial big data ecosystem. You'll understand when and why to use Sedona, and get your development environment set up.

---

## 1.1 What is Apache Sedona? (15 minutes)

### Definition
Apache Sedona (formerly GeoSpark) is a distributed computing framework for processing large-scale spatial data. It extends Apache Spark with specialized capabilities for geospatial operations.

### Key Features
- **Scalability**: Process billions of geometries across clusters
- **SQL Support**: Spatial SQL queries through Spark SQL
- **Multiple APIs**: RDD, DataFrame, and SQL interfaces
- **Spatial Indexing**: R-Tree, Quad-Tree for optimized queries
- **Visualization**: Built-in map generation capabilities
- **Format Support**: Shapefile, GeoJSON, WKT, WKB, GeoParquet
- **Language Support**: Scala, Java, Python, R

### Why Sedona?
Traditional geospatial tools struggle with:
- **Volume**: Datasets with millions/billions of geometries
- **Velocity**: Real-time or near-real-time processing
- **Variety**: Multiple formats and sources

Sedona solves this by leveraging Spark's distributed computing model.

---

## 1.2 History and Evolution (10 minutes)

### Timeline
- **2016**: Started as GeoSpark research project
- **2017**: First public release
- **2020**: Accepted into Apache Incubator
- **2021**: Renamed to Apache Sedona
- **2023**: Graduated as Apache Top-Level Project

### Current Status
- Active development community
- Production use at major companies
- Regular releases with new features
- Growing ecosystem of tools

---

## 1.3 Architecture Overview (15 minutes)

### Core Components

```
┌─────────────────────────────────────────────┐
│         Apache Sedona                        │
├─────────────────────────────────────────────┤
│  Sedona SQL  │  Sedona Python  │  Sedona R  │
├─────────────────────────────────────────────┤
│           Sedona Core (Scala/Java)          │
│  - Spatial RDDs                             │
│  - Spatial Operators                         │
│  - Spatial Indexes                           │
│  - Spatial Partitioning                      │
├─────────────────────────────────────────────┤
│           JTS Topology Suite                 │
│        (Geometry Processing)                 │
├─────────────────────────────────────────────┤
│            Apache Spark                      │
│  - RDD │ DataFrame │ SQL │ Streaming         │
└─────────────────────────────────────────────┘
```

### Layer Breakdown

**1. JTS Topology Suite**
- Java library for geometry operations
- Implements OGC Simple Features specification
- Handles geometric calculations

**2. Sedona Core**
- Spatial RDD implementations
- Distributed spatial operators
- Index structures (R-Tree, Quad-Tree)
- Partitioning schemes

**3. Sedona SQL**
- Spatial SQL functions
- DataFrame operations
- Catalyst optimizer integration

**4. Language Bindings**
- PySedona for Python
- SedonaR for R
- Native Scala/Java APIs

---

## 1.4 Use Cases and Applications (10 minutes)

### Industry Applications

**1. Transportation and Logistics**
- Route optimization
- Delivery zone analysis
- Traffic pattern analysis
- Fleet management

**2. Urban Planning**
- Land use analysis
- Infrastructure planning
- Environmental impact studies
- Zoning compliance

**3. Telecommunications**
- Network coverage analysis
- Tower placement optimization
- Signal strength mapping

**4. Retail and Marketing**
- Location intelligence
- Market basket analysis with location
- Store placement optimization
- Customer proximity analysis

**5. Environmental Science**
- Climate change modeling
- Deforestation tracking
- Wildlife habitat analysis
- Disaster response

**6. Real Estate**
- Property valuation
- Neighborhood analysis
- Development site selection

---

## 1.5 Comparison with Other Tools (5 minutes)

| Feature | Apache Sedona | PostGIS | GeoPandas |
|---------|---------------|---------|-----------|
| Scale | Billions of geometries | Millions | Thousands-Millions |
| Distribution | Yes | No | No |
| SQL Support | Yes | Yes | No |
| Streaming | Yes | No | No |
| In-Memory | Yes | Partial | Yes |
| Visualization | Yes | No | Yes |
| ML Integration | Spark MLlib | Limited | scikit-learn |
| Best For | Big data | Traditional DB | Desktop/Small data |

### When to Use Sedona
- Dataset > 10M geometries
- Need distributed processing
- Integration with Spark pipelines
- Real-time spatial analytics
- Existing Spark infrastructure

### When NOT to Use Sedona
- Small datasets (< 1M rows)
- Simple one-off queries
- No Spark infrastructure
- Desktop GIS analysis

---

## 1.6 Installation and Setup (5 minutes)

### Prerequisites
```bash
# Java 8 or 11
java -version

# Apache Spark 3.0+
spark-submit --version

# Python 3.7+ (for PySedona)
python --version
```

### Installation Options

**Option 1: PySedona (Python)**
```bash
pip install apache-sedona
```

**Option 2: Maven (Scala/Java)**
```xml
<dependency>
    <groupId>org.apache.sedona</groupId>
    <artifactId>sedona-spark-shaded-3.0_2.12</artifactId>
    <version>1.5.1</version>
</dependency>
```

**Option 3: Spark Package**
```bash
pyspark --packages org.apache.sedona:sedona-spark-shaded-3.0_2.12:1.5.1,org.datasyslab:geotools-wrapper:1.5.1-28.2
```

**Option 4: R with SparkR**
```r
install.packages("SparkR")
# Sedona packages loaded when starting Spark session (see verification below)
```

### Verification (Python)
```python
from sedona.spark import *

config = SedonaContext.builder() \
    .appName('sedona-test') \
    .getOrCreate()

sedona = SedonaContext.create(config)
print("Sedona version:", sedona.version)
```

### Verification (R)
```r
library(SparkR)

# Start Spark with Sedona packages
sparkR.session(
  appName = "sedona-test",
  sparkPackages = c(
    "org.apache.sedona:sedona-spark-shaded-3.0_2.12:1.5.1",
    "org.datasyslab:geotools-wrapper:1.5.1-28.2"
  )
)

# Test Sedona SQL
result <- sql("SELECT ST_AsText(ST_Point(0, 0)) as point")
showDF(result)
```

---

## Hands-on Exercise 1: Setup and First Query

### Objective
Set up Sedona and execute your first spatial query.

### Tasks
1. Install Apache Sedona
2. Start Spark session with Sedona
3. Create simple point geometries
4. Perform a spatial operation

### Code Example
```python
from sedona.spark import *
from sedona.core.geom.envelope import Envelope

# Initialize Sedona
config = SedonaContext.builder() \
    .appName('module1-exercise') \
    .master('local[*]') \
    .getOrCreate()

sedona = SedonaContext.create(config)

# Create sample data
data = [
    ("Point 1", "POINT (30 10)"),
    ("Point 2", "POINT (25 15)"),
    ("Point 3", "POINT (35 12)")
]

df = sedona.createDataFrame(data, ["name", "geometry"])

# Register spatial functions
df.createOrReplaceTempView("points")

# Spatial query
result = sedona.sql("""
    SELECT name, ST_AsText(ST_GeomFromWKT(geometry)) as geom
    FROM points
""")

result.show()
```

### R Version

```r
library(SparkR)

# Initialize Sedona
sparkR.session(
  appName = 'module1-exercise',
  master = 'local[*]',
  sparkPackages = c(
    "org.apache.sedona:sedona-spark-shaded-3.0_2.12:1.5.1",
    "org.datasyslab:geotools-wrapper:1.5.1-28.2"
  )
)

# Create sample data
data <- data.frame(
  name = c("Point 1", "Point 2", "Point 3"),
  geometry = c("POINT (30 10)", "POINT (25 15)", "POINT (35 12)")
)

df <- createDataFrame(data)

# Register as temp view
createOrReplaceTempView(df, "points")

# Spatial query
result <- sql("
  SELECT name, ST_AsText(ST_GeomFromWKT(geometry)) as geom
  FROM points
")

showDF(result)
```

### Expected Output
```
+-------+----------------+
|   name|            geom|
+-------+----------------+
|Point 1|  POINT (30 10)|
|Point 2|  POINT (25 15)|
|Point 3|  POINT (35 12)|
+-------+----------------+
```

---

## Key Takeaways

- Apache Sedona enables distributed geospatial processing at scale
- Built on Apache Spark, leveraging its distributed computing capabilities
- Supports multiple languages (Python, Scala, Java, R)
- Ideal for big data geospatial analytics (millions/billions of geometries)
- Provides RDD, DataFrame, and SQL APIs
- Integrates with existing Spark ecosystems

---

## Additional Resources

- [Official Apache Sedona Website](https://sedona.apache.org/)
- [GitHub Repository](https://github.com/apache/sedona)
- [API Documentation](https://sedona.apache.org/latest-snapshot/api/javadoc/)
- [Community Slack Channel](https://sedona-community.slack.com/)

---

## Quiz Questions

1. What was Apache Sedona originally called?
2. Name three industry applications where Sedona is commonly used.
3. What is the maximum scale of geometries PostGIS can efficiently handle compared to Sedona?
4. Which Java library does Sedona use for geometry operations?
5. What are the three main API interfaces Sedona provides?

---

**Next:** [Module 2: Spatial Data Fundamentals](../module2/README.md)
