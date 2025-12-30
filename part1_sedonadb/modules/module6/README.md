# Module 6: Application Development
**Duration:** 1.5 hours

## 6.1 Python Integration (30 min)

```python
import duckdb
import pandas as pd

class SpatialDatabase:
    def __init__(self, db_path=':memory:'):
        self.conn = duckdb.connect(db_path)
        self.conn.execute("INSTALL spatial; LOAD spatial;")

    def load_points(self, df, lon_col='lon', lat_col='lat'):
        """Load DataFrame with coordinates"""
        self.conn.register('temp_df', df)
        self.conn.execute(f"""
            CREATE OR REPLACE TABLE points AS
            SELECT *, ST_Point({lon_col}, {lat_col}) as geom
            FROM temp_df
        """)

    def nearest_neighbors(self, lon, lat, k=5):
        """Find k nearest points"""
        return self.conn.execute(f"""
            SELECT *,
                ST_Distance(geom, ST_Point({lon}, {lat})) as distance
            FROM points
            ORDER BY distance
            LIMIT {k}
        """).df()

    def points_in_radius(self, lon, lat, radius_km):
        """Find points within radius"""
        radius_deg = radius_km / 111.0
        return self.conn.execute(f"""
            SELECT *
            FROM points
            WHERE ST_DWithin(geom, ST_Point({lon}, {lat}), {radius_deg})
        """).df()

# Usage
db = SpatialDatabase('app.duckdb')
df = pd.DataFrame({
    'name': ['A', 'B', 'C'],
    'lon': [-74.0, -73.9, -74.1],
    'lat': [40.7, 40.8, 40.6]
})
db.load_points(df)
nearest = db.nearest_neighbors(-74.0, 40.7, k=2)
print(nearest)
```

## 6.2 REST API with FastAPI (25 min)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import duckdb

app = FastAPI()
conn = duckdb.connect('spatial_api.duckdb')
conn.execute("INSTALL spatial; LOAD spatial;")

class Point(BaseModel):
    lon: float
    lat: float

class SearchRequest(BaseModel):
    point: Point
    radius_km: float
    limit: int = 10

@app.get("/places/nearest")
async def get_nearest(lon: float, lat: float, limit: int = 5):
    result = conn.execute(f"""
        SELECT name, category,
            ST_Distance(location, ST_Point({lon}, {lat})) * 111 as distance_km
        FROM places
        ORDER BY distance_km
        LIMIT {limit}
    """).df()
    return result.to_dict(orient='records')

@app.post("/places/search")
async def search_radius(request: SearchRequest):
    radius_deg = request.radius_km / 111.0
    result = conn.execute(f"""
        SELECT *
        FROM places
        WHERE ST_DWithin(
            location,
            ST_Point({request.point.lon}, {request.point.lat}),
            {radius_deg}
        )
        LIMIT {request.limit}
    """).df()
    return result.to_dict(orient='records')

# Run with: uvicorn app:app --reload
```

## 6.3 Web Mapping Integration (20 min)

```python
import duckdb
import folium

conn = duckdb.connect('mapping.duckdb')
conn.execute("INSTALL spatial; LOAD spatial;")

# Get data
locations = conn.execute("""
    SELECT name, ST_X(geom) as lon, ST_Y(geom) as lat, category
    FROM locations
""").df()

# Create map
m = folium.Map(location=[locations['lat'].mean(), locations['lon'].mean()], zoom_start=12)

# Add markers
for idx, row in locations.iterrows():
    folium.Marker(
        [row['lat'], row['lon']],
        popup=row['name'],
        tooltip=row['category']
    ).add_to(m)

# Save
m.save('map.html')
```

## 6.4 R Integration (15 min)

```r
library(duckdb)
library(sf)

con <- dbConnect(duckdb::duckdb(), "spatial.duckdb")
dbExecute(con, "INSTALL spatial; LOAD spatial;")

# Load spatial data
dbExecute(con, "
    CREATE TABLE locations AS
    SELECT * FROM ST_Read('data.geojson')
")

# Query and convert to sf
result <- dbGetQuery(con, "
    SELECT name, ST_AsText(geom) as geometry
    FROM locations
")

# Convert to sf object
result_sf <- st_as_sf(result, wkt = "geometry")
plot(result_sf)
```

---

**Previous:** [Module 5](../module5/README.md) | **Next:** [Module 7](../module7/README.md)
