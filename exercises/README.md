# Hands-on Exercises and Final Project

## Overview
This section contains practical exercises and a comprehensive final project to reinforce your Apache Sedona skills.

---

## Practice Exercises

### Exercise 1: Setup Verification
**Duration:** 15 minutes

**Objective:** Verify your Sedona installation and create your first spatial query.

**Tasks:**
1. Install Apache Sedona
2. Start a Sedona session
3. Create sample geometries
4. Execute basic spatial functions

**Starter Code:**
```python
from sedona.spark import *

# Initialize Sedona
config = SedonaContext.builder() \
    .appName('exercise1') \
    .master('local[*]') \
    .getOrCreate()

sedona = SedonaContext.create(config)

# Your code here
# 1. Create a DataFrame with 5 points
# 2. Calculate distance between points
# 3. Create buffer around each point
```

---

### Exercise 2: Data Loading Challenge
**Duration:** 30 minutes

**Objective:** Load spatial data from multiple formats and transform CRS.

**Tasks:**
1. Load CSV with coordinates
2. Load GeoJSON file
3. Transform to common CRS (3857)
4. Join datasets spatially
5. Export results

**Dataset:** Download sample data from `/datasets/exercise2/`

---

### Exercise 3: Spatial Query Optimization
**Duration:** 45 minutes

**Objective:** Compare query performance with and without optimizations.

**Tasks:**
1. Load 1M point dataset
2. Perform spatial join WITHOUT indexing
3. Measure execution time
4. Apply spatial partitioning
5. Build R-Tree index
6. Re-run query and compare performance

**Expected Result:** 10-50x speedup

---

### Exercise 4: Real-time Geofencing
**Duration:** 1 hour

**Objective:** Build a streaming geofence monitoring system.

**Tasks:**
1. Set up Kafka topic (or use file stream)
2. Define geofence polygons
3. Stream location updates
4. Detect geofence entry/exit events
5. Aggregate events by time window

**Bonus:** Add alerting for specific geofences

---

### Exercise 5: Heatmap Generation
**Duration:** 45 minutes

**Objective:** Create density heatmap from point data.

**Tasks:**
1. Load point dataset (taxi pickups, crime incidents, etc.)
2. Create grid-based heatmap
3. Identify top 10 hotspots
4. Visualize results
5. Export for web visualization

---

## Final Project: Urban Mobility Analysis Platform

### Project Overview
Build a comprehensive urban mobility analysis system that processes large-scale transportation data to provide insights for city planning.

### Duration
3-4 hours

### Datasets Required
1. **Transit routes** (LineString) - Bus/metro routes
2. **Transit stops** (Point) - Station locations
3. **Trip records** (Point) - Origin-destination pairs
4. **Census blocks** (Polygon) - Demographic data
5. **Points of Interest** (Point) - Major destinations
6. **Administrative boundaries** (Polygon) - City districts

### Business Requirements

**Part 1: Data Integration (30 minutes)**
- Load all datasets from various formats
- Standardize to common CRS (Web Mercator 3857)
- Validate geometries
- Create unified data model

**Part 2: Transit Accessibility Analysis (45 minutes)**
- Calculate 400m walking distance buffers around each transit stop
- Identify census blocks within transit coverage
- Compute population served by transit
- Find underserved areas (low transit coverage, high population)

**Part 3: Trip Pattern Analysis (45 minutes)**
- Analyze trip origins and destinations
- Identify high-volume corridors
- Create OD (origin-destination) matrix by district
- Detect peak hour patterns

**Part 4: Service Gap Analysis (45 minutes)**
- Find areas with high POI density but low transit coverage
- Identify potential new transit route locations
- Calculate optimal stop locations using spatial clustering
- Estimate ridership for proposed routes

**Part 5: Visualization and Reporting (30 minutes)**
- Create heatmaps of trip density
- Generate coverage maps
- Export results for web visualization
- Create summary statistics report

### Deliverables

**Code:**
```
final_project/
├── notebooks/
│   ├── 01_data_loading.ipynb
│   ├── 02_transit_coverage.ipynb
│   ├── 03_trip_analysis.ipynb
│   ├── 04_service_gaps.ipynb
│   └── 05_visualization.ipynb
├── src/
│   ├── data_loader.py
│   ├── spatial_analysis.py
│   ├── visualization.py
│   └── utils.py
├── output/
│   ├── transit_coverage.geojson
│   ├── trip_heatmap.geojson
│   ├── service_gaps.geojson
│   └── summary_report.csv
└── README.md
```

**Report:**
- Executive summary (1 page)
- Methodology description
- Key findings with visualizations
- Recommendations for city planners
- Technical appendix (code snippets, performance metrics)

### Evaluation Criteria

**Technical Implementation (60%)**
- [ ] Correct use of Sedona spatial functions
- [ ] Proper CRS handling
- [ ] Efficient query optimization
- [ ] Code organization and documentation
- [ ] Error handling

**Analysis Quality (30%)**
- [ ] Meaningful insights derived
- [ ] Appropriate analytical methods
- [ ] Clear visualizations
- [ ] Actionable recommendations

**Presentation (10%)**
- [ ] Clear documentation
- [ ] Professional report format
- [ ] Reproducible results

### Starter Template

```python
"""
Urban Mobility Analysis Platform
Final Project - Apache Sedona Course
"""

from sedona.spark import *
from pyspark.sql.functions import expr, count, sum as sql_sum, avg

class MobilityAnalyzer:
    def __init__(self):
        """Initialize Sedona context"""
        config = SedonaContext.builder() \
            .appName('urban-mobility-analysis') \
            .config("spark.executor.memory", "4g") \
            .config("spark.driver.memory", "2g") \
            .getOrCreate()

        self.sedona = SedonaContext.create(config)

    def load_data(self):
        """Load and standardize all datasets"""
        # TODO: Implement data loading
        pass

    def analyze_transit_coverage(self):
        """Calculate transit accessibility metrics"""
        # TODO: Implement transit coverage analysis
        pass

    def analyze_trip_patterns(self):
        """Analyze origin-destination patterns"""
        # TODO: Implement trip pattern analysis
        pass

    def identify_service_gaps(self):
        """Find areas needing better transit service"""
        # TODO: Implement service gap analysis
        pass

    def generate_visualizations(self):
        """Create maps and charts"""
        # TODO: Implement visualization generation
        pass

    def generate_report(self):
        """Create summary report"""
        # TODO: Implement report generation
        pass

if __name__ == "__main__":
    analyzer = MobilityAnalyzer()

    # Execute analysis pipeline
    print("Loading data...")
    analyzer.load_data()

    print("Analyzing transit coverage...")
    analyzer.analyze_transit_coverage()

    print("Analyzing trip patterns...")
    analyzer.analyze_trip_patterns()

    print("Identifying service gaps...")
    analyzer.identify_service_gaps()

    print("Generating visualizations...")
    analyzer.generate_visualizations()

    print("Creating report...")
    analyzer.generate_report()

    print("Analysis complete!")
```

### Sample Solutions

Complete solutions for all exercises and the final project are available in the `/solutions/` directory. Try to complete the exercises on your own before consulting the solutions.

---

## Bonus Challenges

### Challenge 1: Performance Optimization
Take the final project and optimize it for:
- Dataset size: 100M+ records
- Query latency: <10 seconds
- Memory usage: <16GB

### Challenge 2: Real-time Dashboard
Extend the final project to:
- Process streaming trip data
- Update metrics in real-time
- Display live heatmap
- Alert on anomalies

### Challenge 3: Multi-Modal Analysis
Add additional transportation modes:
- Bike-sharing stations and trips
- Ride-sharing pickups/dropoffs
- Pedestrian counts
- Calculate multi-modal accessibility

---

## Dataset Sources

### Public Datasets You Can Use

**Transportation:**
- NYC Taxi & Limousine Commission Trip Records
- GTFS (General Transit Feed Specification) data
- OpenStreetMap road networks

**Geographic:**
- US Census Bureau TIGER/Line Shapefiles
- Natural Earth datasets
- OpenStreetMap building footprints

**Points of Interest:**
- OpenStreetMap POI data
- Foursquare Places API (with free tier)

**Demographic:**
- US Census ACS (American Community Survey)
- WorldPop population density

### Generating Synthetic Data

```python
import random
from shapely.geometry import Point, Polygon
import pandas as pd

def generate_points(n, bbox):
    """Generate random points within bounding box"""
    points = []
    for i in range(n):
        x = random.uniform(bbox[0], bbox[2])
        y = random.uniform(bbox[1], bbox[3])
        points.append({
            'id': i,
            'lon': x,
            'lat': y,
            'value': random.randint(1, 100)
        })
    return pd.DataFrame(points)

# Generate 100K random points for NYC
nyc_bbox = (-74.05, 40.68, -73.90, 40.88)
points_df = generate_points(100000, nyc_bbox)
points_df.to_csv('datasets/generated_points.csv', index=False)
```

---

## Getting Help

**Stuck on an exercise?**
1. Review the relevant module
2. Check the Sedona documentation
3. Look at code examples in modules
4. Ask in discussion forums
5. Consult solution (last resort!)

**Performance issues?**
1. Check Spark UI for bottlenecks
2. Review Module 5 (Indexing and Partitioning)
3. Review Module 7 (Best Practices)
4. Start with small sample data
5. Scale up gradually

**Data issues?**
1. Validate geometries with ST_IsValid
2. Check CRS with ST_SRID
3. Verify data types
4. Look for nulls
5. Check data ranges

---

## Certification

Upon completing all exercises and the final project, you'll have:
- Hands-on experience with all major Sedona features
- Production-ready code samples
- A portfolio project for your resume
- Deep understanding of distributed geospatial processing

**Next Steps:**
- Apply Sedona to your own datasets
- Contribute to Apache Sedona project
- Explore advanced features (Spark MLlib integration, etc.)
- Share your learnings with the community

---

**Back to:** [Course Outline](../COURSE_OUTLINE.md) | [Module 7](../modules/module7/README.md)
