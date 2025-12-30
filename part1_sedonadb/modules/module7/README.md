# Module 7: Production Deployment and Case Studies
**Duration:** 1.5 hours

## 7.1 Case Study 1: Location-Based Service (20 min)

### Requirements
- Restaurant finder app
- 100K+ restaurants
- Real-time nearest search
- < 100ms query latency

### Solution

```python
import duckdb
from fastapi import FastAPI

class RestaurantFinder:
    def __init__(self):
        self.conn = duckdb.connect('restaurants.duckdb')
        self.conn.execute("INSTALL spatial; LOAD spatial;")
        self._setup_indexes()

    def _setup_indexes(self):
        self.conn.execute("""
            CREATE INDEX IF NOT EXISTS idx_location
            ON restaurants USING RTREE (location);

            CREATE INDEX IF NOT EXISTS idx_cuisine
            ON restaurants(cuisine);

            CREATE INDEX IF NOT EXISTS idx_rating
            ON restaurants(rating);
        """)

    def find_nearby(self, lon, lat, radius_km=5, cuisine=None, min_rating=3.0):
        radius_deg = radius_km / 111.0
        query = f"""
            SELECT
                name,
                cuisine,
                rating,
                ST_Distance(location, ST_Point({lon}, {lat})) * 111 as distance_km
            FROM restaurants
            WHERE ST_DWithin(location, ST_Point({lon}, {lat}), {radius_deg})
                AND rating >= {min_rating}
        """
        if cuisine:
            query += f" AND cuisine = '{cuisine}'"
        query += " ORDER BY distance_km LIMIT 20"

        return self.conn.execute(query).df()

# Deploy with Docker
# FROM python:3.11
# RUN pip install duckdb fastapi uvicorn
# COPY app.py .
# CMD ["uvicorn", "app:app", "--host", "0.0.0.0"]
```

## 7.2 Case Study 2: Real Estate Search (20 min)

### Requirements
- Property search by location
- Filter by price, size, type
- Nearby amenities scoring

### Solution

```sql
-- Create comprehensive property view
CREATE VIEW property_search AS
WITH property_amenities AS (
    SELECT
        p.id as property_id,
        COUNT(CASE WHEN a.type = 'school' THEN 1 END) as schools_within_1km,
        COUNT(CASE WHEN a.type = 'park' THEN 1 END) as parks_within_1km,
        COUNT(CASE WHEN a.type = 'transit' THEN 1 END) as transit_within_500m
    FROM properties p
    LEFT JOIN amenities a
        ON (a.type IN ('school', 'park') AND ST_DWithin(p.location, a.location, 0.01))
        OR (a.type = 'transit' AND ST_DWithin(p.location, a.location, 0.005))
    GROUP BY p.id
)
SELECT
    p.*,
    pa.schools_within_1km,
    pa.parks_within_1km,
    pa.transit_within_500m,
    (pa.schools_within_1km * 10 + pa.parks_within_1km * 5 + pa.transit_within_500m * 15) as amenity_score
FROM properties p
LEFT JOIN property_amenities pa ON p.id = pa.property_id;

-- Materialized for performance
CREATE TABLE property_search_materialized AS SELECT * FROM property_search;
CREATE INDEX idx_price ON property_search_materialized(price);
CREATE INDEX idx_score ON property_search_materialized(amenity_score);
```

## 7.3 Case Study 3: Urban Planning Tool (20 min)

### Analytics Pipeline

```python
import duckdb

class UrbanAnalytics:
    def __init__(self, db_path='urban.duckdb'):
        self.conn = duckdb.connect(db_path)
        self.conn.execute("INSTALL spatial; LOAD spatial;")

    def load_city_data(self, buildings_file, parcels_file, zones_file):
        """Load all spatial datasets"""
        self.conn.execute(f"CREATE TABLE buildings AS SELECT * FROM ST_Read('{buildings_file}')")
        self.conn.execute(f"CREATE TABLE parcels AS SELECT * FROM ST_Read('{parcels_file}')")
        self.conn.execute(f"CREATE TABLE zones AS SELECT * FROM ST_Read('{zones_file}')")

    def analyze_density(self):
        """Calculate building density by zone"""
        return self.conn.execute("""
            SELECT
                z.zone_name,
                z.zone_type,
                COUNT(b.id) as building_count,
                SUM(b.floor_area) as total_floor_area,
                ST_Area(z.geom) as zone_area,
                SUM(b.floor_area) / ST_Area(z.geom) as far_ratio
            FROM zones z
            LEFT JOIN buildings b ON ST_Contains(z.geom, b.geom)
            GROUP BY z.zone_name, z.zone_type, z.geom
            ORDER BY far_ratio DESC
        """).df()

    def find_development_sites(self, min_area_sqm=1000):
        """Find potential development sites"""
        return self.conn.execute(f"""
            SELECT
                p.parcel_id,
                ST_Area(p.geom) as area_sqm,
                z.zone_type,
                z.max_height,
                CASE
                    WHEN b.id IS NULL THEN 'Vacant'
                    ELSE 'Underutilized'
                END as status
            FROM parcels p
            JOIN zones z ON ST_Contains(z.geom, p.geom)
            LEFT JOIN buildings b ON ST_Contains(p.geom, b.geom)
            WHERE ST_Area(p.geom) >= {min_area_sqm}
                AND (b.id IS NULL OR b.floor_area / ST_Area(p.geom) < 0.5)
            ORDER BY area_sqm DESC
        """).df()
```

## 7.4 Production Deployment (20 min)

### Docker Deployment

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
RUN pip install duckdb fastapi uvicorn python-multipart

# Copy application
COPY app.py .
COPY spatial_db.duckdb .

# Expose port
EXPOSE 8000

# Run
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Backup Strategy

```python
import duckdb
import shutil
from datetime import datetime

def backup_database(source_db, backup_dir):
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    backup_path = f"{backup_dir}/backup_{timestamp}.duckdb"

    # Copy database file
    shutil.copy2(source_db, backup_path)

    # Export critical tables to Parquet
    conn = duckdb.connect(source_db)
    conn.execute(f"COPY (SELECT * FROM spatial_table) TO '{backup_dir}/spatial_{timestamp}.parquet'")
    conn.close()

    return backup_path
```

### Monitoring

```python
import psutil
import duckdb

def check_database_health(db_path):
    conn = duckdb.connect(db_path)

    # Check table sizes
    tables = conn.execute("""
        SELECT table_name, estimated_size
        FROM duckdb_tables()
        WHERE schema_name = 'main'
    """).df()

    # Check system resources
    memory = psutil.virtual_memory()
    disk = psutil.disk_usage('/')

    return {
        'tables': tables.to_dict('records'),
        'memory_available_gb': memory.available / (1024**3),
        'disk_free_gb': disk.free / (1024**3)
    }
```

## 7.5 Migration to Distributed Sedona (10 min)

### When to Migrate
- Database size > 500GB
- Query times > 5 minutes
- Need horizontal scaling
- Real-time streaming requirements

### Migration Steps

```python
# 1. Export from SedonaDB
import duckdb
conn = duckdb.connect('sedonadb.duckdb')
conn.execute("COPY spatial_table TO 'export.parquet' (FORMAT PARQUET)")

# 2. Load into Distributed Sedona
from sedona.spark import *

config = SedonaContext.builder().appName('migration').getOrCreate()
sedona = SedonaContext.create(config)

# Read exported parquet
df = sedona.read.format("geoparquet").load("export.parquet")

# Continue with Part 2 course!
```

---

## Key Takeaways

- SedonaDB excels for single-node spatial applications
- Production deployments require indexing, monitoring, backups
- Real-world use cases: location services, real estate, urban planning
- Docker simplifies deployment
- Clear migration path to Distributed Sedona when scaling needed

---

## Transition to Part 2

You now have a solid foundation in single-node geospatial databases. **Part 2: Distributed Sedona** teaches you to scale beyond single-node limitations using Apache Spark for processing billions of geometries across clusters.

**Continue to:** [Part 2: Distributed Sedona](../../part2_distributed/README.md)

---

**Previous:** [Module 6](../module6/README.md) | **Back to Course:** [Part 1 Overview](../../part1_sedonadb/README.md)
