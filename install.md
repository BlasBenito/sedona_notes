# Installation Guide
Complete installation instructions for the Apache Sedona course (Parts 1 & 2)

---

## ⚠️ Important: Externally-Managed Environment

**Modern Debian/Ubuntu systems (Python 3.11+) prevent global pip installations.**

You'll see this error:
```
error: externally-managed-environment
× This environment is externally managed
```

**Solution:** Always use a virtual environment (required on Debian/Ubuntu, recommended everywhere).

---

## System Requirements

### Minimum
- **OS**: Linux, macOS, or Windows (WSL2 recommended)
- **RAM**: 8GB minimum, 16GB recommended
- **Disk**: 10GB free space
- **CPU**: 4 cores recommended

### Software Prerequisites
- **Python**: 3.8 or higher
- **Java**: 8 or 11 (required for Part 2)
- **R**: 4.0 or higher (optional, for R users)

**For Debian/Ubuntu users:**
```bash
sudo apt install python3-full python3-venv python3-pip
```

---

## Quick Start (Recommended)

### For Python Users

**Step-by-step:**

```bash
# 1. Install prerequisites (Debian/Ubuntu only)
sudo apt install python3-full python3-venv python3-pip

# 2. Create virtual environment
python3 -m venv ~/sedona-env

# 3. Activate virtual environment
source ~/sedona-env/bin/activate  # On Windows: sedona-env\Scripts\activate

# 4. Upgrade pip
pip install --upgrade pip

# 5. Install for both parts
pip install duckdb apache-sedona pyspark jupyter pandas geopandas folium

# 6. Verify installation
python -c "import duckdb; import pyspark; print('✓ All packages installed')"
```

**Remember:** Always activate the environment before using:
```bash
source ~/sedona-env/bin/activate
```

### For R Users

```r
# Install R packages
install.packages(c("duckdb", "SparkR", "sparklyr", "sf", "ggplot2"))
```

---

## Part 1: SedonaDB Installation

### Python Setup

```bash
# Activate virtual environment
source ~/sedona-env/bin/activate

# Install DuckDB with spatial extension
pip install duckdb

# Optional: Additional packages for Part 1
pip install pandas geopandas folium fastapi uvicorn
```

### Verify Part 1 (Python)

```python
import duckdb

conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial;")
conn.execute("LOAD spatial;")

# Test spatial query
result = conn.execute("SELECT ST_AsText(ST_Point(0, 0))").fetchone()
print(f"✓ SedonaDB working: {result[0]}")
```

### R Setup for Part 1

```r
# Install DuckDB for R
install.packages("duckdb")

# Optional packages
install.packages(c("sf", "ggplot2", "leaflet"))
```

### Verify Part 1 (R)

```r
library(duckdb)

con <- dbConnect(duckdb::duckdb(), ":memory:")
dbExecute(con, "INSTALL spatial;")
dbExecute(con, "LOAD spatial;")

# Test spatial query
result <- dbGetQuery(con, "SELECT ST_AsText(ST_Point(0, 0)) as point")
print(paste("✓ SedonaDB working:", result$point))
```

---

## Part 2: Distributed Sedona Installation

### Step 1: Install Java

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install openjdk-11-jdk
java -version  # Verify
```

**macOS:**
```bash
brew install openjdk@11
echo 'export PATH="/usr/local/opt/openjdk@11/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
java -version  # Verify
```

**Windows:**
Download and install from [Oracle](https://www.oracle.com/java/technologies/downloads/) or [OpenJDK](https://adoptium.net/)

### Step 2: Install Apache Spark (Standalone)

**Option A: Using pip (Recommended)**
```bash
pip install pyspark==3.4.0
```

**Option B: Manual Installation**
```bash
# Download Spark
wget https://archive.apache.org/dist/spark/spark-3.4.0/spark-3.4.0-bin-hadoop3.tgz
tar -xzf spark-3.4.0-bin-hadoop3.tgz
sudo mv spark-3.4.0-bin-hadoop3 /opt/spark

# Set environment variables
echo 'export SPARK_HOME=/opt/spark' >> ~/.bashrc
echo 'export PATH=$SPARK_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Verify
spark-submit --version
```

### Step 3: Install Apache Sedona

**Python:**
```bash
# Activate virtual environment
source ~/sedona-env/bin/activate

# Install Apache Sedona
pip install apache-sedona

# Install additional dependencies
pip install pyspark==3.4.0 geopandas shapely
```

**R:**
```r
# Install SparkR (comes with Spark)
install.packages("SparkR")

# Or sparklyr
install.packages("sparklyr")

# Install Spark using sparklyr
library(sparklyr)
spark_install(version = "3.4.0")
```

### Step 4: Verify Part 2 (Python)

```python
from sedona.spark import *

config = SedonaContext.builder() \
    .appName("test") \
    .master("local[*]") \
    .config("spark.executor.memory", "2g") \
    .config("spark.driver.memory", "2g") \
    .getOrCreate()

sedona = SedonaContext.create(config)

# Test spatial query
result = sedona.sql("SELECT ST_AsText(ST_Point(0, 0)) as point")
result.show()
print("✓ Distributed Sedona working!")

# Stop session
sedona.stop()
```

### Step 5: Verify Part 2 (R)

```r
library(SparkR)

# Start Spark with Sedona
sparkR.session(
  appName = "test",
  master = "local[*]",
  sparkPackages = c(
    "org.apache.sedona:sedona-spark-shaded-3.0_2.12:1.5.1",
    "org.datasyslab:geotools-wrapper:1.5.1-28.2"
  ),
  sparkConfig = list(
    spark.executor.memory = "2g",
    spark.driver.memory = "2g"
  )
)

# Test spatial query
result <- sql("SELECT ST_AsText(ST_Point(0, 0)) as point")
showDF(result)
print("✓ Distributed Sedona working!")

# Stop session
sparkR.session.stop()
```

---

## Complete Installation Script (Linux/macOS)

### Bash Script for Everything

```bash
#!/bin/bash
set -e

echo "=== Installing Apache Sedona Course Dependencies ==="

# 1. Install system prerequisites (Debian/Ubuntu)
if [[ "$OSTYPE" == "linux-gnu"* ]]; then
    echo "Installing Python prerequisites for Debian/Ubuntu..."
    sudo apt update
    sudo apt install -y python3-full python3-venv python3-pip
fi

# 2. Check system requirements
echo "Checking system requirements..."
python3 --version || { echo "Python 3.8+ required"; exit 1; }

# 3. Create virtual environment
echo "Creating virtual environment..."
python3 -m venv ~/sedona-env
source ~/sedona-env/bin/activate

# 3. Upgrade pip
echo "Upgrading pip..."
pip install --upgrade pip

# 4. Install Part 1 dependencies (SedonaDB)
echo "Installing Part 1 (SedonaDB) dependencies..."
pip install duckdb pandas geopandas folium fastapi uvicorn jupyter

# 5. Install Java (if not present)
if ! command -v java &> /dev/null; then
    echo "Installing Java 11..."
    if [[ "$OSTYPE" == "linux-gnu"* ]]; then
        sudo apt update && sudo apt install -y openjdk-11-jdk
    elif [[ "$OSTYPE" == "darwin"* ]]; then
        brew install openjdk@11
    fi
fi

# 6. Install Part 2 dependencies (Distributed Sedona)
echo "Installing Part 2 (Distributed Sedona) dependencies..."
pip install pyspark==3.4.0 apache-sedona shapely

# 7. Install Jupyter for notebooks
echo "Installing Jupyter..."
pip install jupyter notebook

# 8. Verify installations
echo ""
echo "=== Verification ==="

# Verify Part 1
python3 << EOF
import duckdb
conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial; LOAD spatial;")
result = conn.execute("SELECT ST_AsText(ST_Point(0, 0))").fetchone()
print(f"✓ Part 1 (SedonaDB): {result[0]}")
EOF

# Verify Part 2
python3 << EOF
from sedona.spark import *
config = SedonaContext.builder().appName("test").master("local[2]").getOrCreate()
sedona = SedonaContext.create(config)
result = sedona.sql("SELECT ST_AsText(ST_Point(0, 0)) as point")
print("✓ Part 2 (Distributed Sedona): Working!")
sedona.stop()
EOF

echo ""
echo "=== Installation Complete! ==="
echo "To activate environment: source ~/sedona-env/bin/activate"
echo "To start Jupyter: jupyter notebook"
```

**Save as `install_all.sh` and run:**
```bash
chmod +x install_all.sh
./install_all.sh
```

---

## Docker Installation (Alternative)

### Using Docker

```dockerfile
FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    openjdk-11-jdk \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Set Java environment
ENV JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64

# Install Python packages
RUN pip install --upgrade pip && \
    pip install \
    duckdb \
    apache-sedona \
    pyspark==3.4.0 \
    pandas \
    geopandas \
    folium \
    jupyter \
    notebook

WORKDIR /workspace

CMD ["jupyter", "notebook", "--ip=0.0.0.0", "--allow-root", "--no-browser"]
```

**Build and run:**
```bash
docker build -t sedona-course .
docker run -p 8888:8888 -v $(pwd):/workspace sedona-course
```

---

## Package Versions (Tested)

```
Python: 3.11.x
Java: 11.0.x
DuckDB: 0.10.x
Apache Sedona: 1.5.1
PySpark: 3.4.0
Pandas: 2.x
GeoPandas: 0.14.x
Shapely: 2.x
```

---

## Environment Variables

### For Spark (Part 2)

Add to `~/.bashrc` or `~/.zshrc`:

```bash
# Java
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64  # Adjust path
export PATH=$JAVA_HOME/bin:$PATH

# Spark (if manually installed)
export SPARK_HOME=/opt/spark
export PATH=$SPARK_HOME/bin:$PATH
export PYSPARK_PYTHON=python3

# Optional: Set memory limits
export SPARK_DRIVER_MEMORY=4g
export SPARK_EXECUTOR_MEMORY=4g
```

Then reload:
```bash
source ~/.bashrc  # or source ~/.zshrc
```

---

## Jupyter Notebook Setup

```bash
# Activate environment
source ~/sedona-env/bin/activate

# Install Jupyter kernel
pip install ipykernel
python -m ipykernel install --user --name sedona --display-name "Sedona Course"

# Start Jupyter
jupyter notebook
```

In Jupyter, select kernel: **Sedona Course**

---

## Troubleshooting

### Issue: "externally-managed-environment" error

**Problem:**
```
error: externally-managed-environment
× This environment is externally managed
```

**This affects:** Debian, Ubuntu, and derivatives with Python 3.11+

**Solution 1: Use Virtual Environment (Recommended)**
```bash
# Create virtual environment
python3 -m venv ~/sedona-env

# Activate it
source ~/sedona-env/bin/activate

# Now pip works normally
pip install apache-sedona
```

**Solution 2: Use pipx for tools**
```bash
# Install pipx
sudo apt install pipx
pipx ensurepath

# Install with pipx (for command-line tools only)
pipx install jupyter

# Note: For this course, use virtual environment instead
```

**Solution 3: System packages (Limited)**
```bash
# Only for packages available in apt
sudo apt install python3-pandas python3-numpy

# Note: Sedona packages NOT available via apt
# Must use virtual environment
```

**❌ NOT Recommended:**
```bash
# DO NOT DO THIS - breaks system Python
pip install --break-system-packages apache-sedona
```

**Why this happens:**
- PEP 668: Prevents conflicts between pip and system package manager
- Protects system Python from breaking
- Virtual environments are the safe solution

---

### Issue: Java not found

```bash
# Check Java installation
java -version

# If not found, install:
# Ubuntu/Debian:
sudo apt install openjdk-11-jdk

# macOS:
brew install openjdk@11
```

### Issue: "No module named 'pyspark'"

**First, make sure virtual environment is activated:**
```bash
source ~/sedona-env/bin/activate
```

**Then install:**
```bash
pip install pyspark==3.4.0
```

**To verify activation:**
```bash
which python  # Should show ~/sedona-env/bin/python
```

### Issue: Memory errors in Spark

```python
# Reduce memory requirements
config = SedonaContext.builder() \
    .appName("test") \
    .config("spark.executor.memory", "1g") \
    .config("spark.driver.memory", "1g") \
    .getOrCreate()
```

### Issue: DuckDB spatial extension not found

```python
import duckdb
conn = duckdb.connect(':memory:')
conn.execute("INSTALL spatial;")  # Install extension
conn.execute("LOAD spatial;")     # Load extension
```

### Issue: R SparkR package not found

```r
# Install from CRAN
install.packages("SparkR")

# Or from Spark installation
install.packages("/opt/spark/R/lib/SparkR", repos = NULL, type = "source")
```

### Issue: Permission denied (Linux)

```bash
# Add user to necessary groups
sudo usermod -aG sudo $USER

# Or use sudo for apt commands
sudo apt install <package>
```

---

## Platform-Specific Notes

### Windows (WSL2 Recommended)

1. Install WSL2: https://docs.microsoft.com/en-us/windows/wsl/install
2. Install Ubuntu from Microsoft Store
3. Follow Linux instructions above inside WSL2

**Alternative (Native Windows):**
- Install Python from python.org
- Install Java from Oracle or Adoptium
- Use PowerShell for installation commands
- Replace `source activate` with `.\sedona-env\Scripts\activate`

### macOS (Apple Silicon M1/M2)

```bash
# Use Rosetta for compatibility
arch -x86_64 brew install openjdk@11
arch -x86_64 pip install apache-sedona
```

### Linux (ARM/Raspberry Pi)

```bash
# Use compatible Java
sudo apt install openjdk-11-jdk

# May need to build from source for some packages
pip install --no-binary :all: apache-sedona
```

---

## Minimal Installation (Part 1 Only)

If you only want Part 1 (SedonaDB):

```bash
python3 -m venv ~/sedona-env
source ~/sedona-env/bin/activate
pip install duckdb jupyter pandas
```

---

## Full Installation (Both Parts)

For complete course experience:

```bash
# Create environment
python3 -m venv ~/sedona-env
source ~/sedona-env/bin/activate

# Install everything
pip install \
    duckdb \
    apache-sedona \
    pyspark==3.4.0 \
    pandas \
    geopandas \
    shapely \
    folium \
    fastapi \
    uvicorn \
    jupyter \
    notebook \
    matplotlib
```

---

## Verification Checklist

Run these commands to verify everything works:

**Part 1:**
```bash
python -c "import duckdb; print('✓ DuckDB installed')"
python -c "import pandas; print('✓ Pandas installed')"
```

**Part 2:**
```bash
python -c "import pyspark; print('✓ PySpark installed')"
python -c "from sedona.spark import *; print('✓ Apache Sedona installed')"
java -version  # Should show Java 11
```

**All checks passing?** You're ready to start the course! 🎉

---

## Next Steps

After installation:

1. **Verify everything works** using commands above
2. **Start with Part 1:** [SedonaDB](part1_sedonadb/README.md)
3. **Clone course materials** (if not already done):
   ```bash
   git clone https://github.com/BlasBenito/sedona_notes.git
   cd sedona_notes
   ```

---

## Getting Help

- Check troubleshooting section above
- Review [Part 1 Module 1](part1_sedonadb/modules/module1/README.md) for SedonaDB setup
- Review [Part 2 Module 1](part2_distributed/modules/module1/README.md) for Distributed Sedona setup
- See [R Guide](part2_distributed/R_GUIDE.md) for R-specific installation

---

**Ready to begin?** → [Start with Part 1](part1_sedonadb/README.md)
