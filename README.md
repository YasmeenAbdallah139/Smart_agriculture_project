# Smart Farming ETL Pipeline - Airflow Implementation

## 📋 Overview

This Airflow DAG automates the ETL (Extract, Transform, Load) process for Smart Farming sensor data. It processes data from HDFS, performs transformations, and loads it into a MySQL data warehouse with **incremental processing** using timestamp checkpoints.

## 🎯 Key Features

- ✅ **Incremental Processing**: Only processes new data since the last run
- ✅ **Automatic Checkpoint Management**: Tracks processed timestamps
- ✅ **Smart Resource Usage**: Skips execution when no new data is available
- ✅ **Data Quality**: Handles missing values and removes outliers
- ✅ **Dimension Modeling**: Creates star schema with fact and dimension tables
- ✅ **Fault Tolerance**: Includes retry logic and proper error handling

## 📁 Project Structure

```
airflow/
├── dags/
│   └── smart_farming_etl_incremental.py  # Main Airflow DAG
├── logs/                                   # Airflow logs
└── /tmp/
    ├── last_processed_timestamp.txt        # Checkpoint file
    ├── spark_transformed_data/             # Temporary transformed data
    └── spark_tables/                       # Temporary dimension/fact tables
```

## 🔧 Prerequisites

### 1. Software Requirements
- **Apache Airflow** (v2.0+)
- **Apache Spark** (v3.0+)
- **PySpark**
- **Hadoop/HDFS**
- **MySQL** (v8.0+)
- **Python** (v3.7+)

### 2. Python Dependencies
```bash
pip install apache-airflow
pip install pyspark
pip install pymysql
pip install sqlalchemy
```

### 3. Airflow Spark Configuration
Add to `airflow.cfg` or set environment variables:
```ini
[spark]
spark_home = /path/to/spark
```

### 4. JDBC Driver
Download MySQL JDBC driver and place it in Spark's `jars` directory:
```bash
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-8.0.33.jar
mv mysql-connector-java-8.0.33.jar $SPARK_HOME/jars/
```

## 🚀 Setup Instructions

### Step 1: Deploy the DAG

1. Copy the DAG file to your Airflow DAGs folder:
```bash
cp smart_farming_etl_incremental.py ~/airflow/dags/
```

2. Verify the DAG is detected:
```bash
airflow dags list | grep smart_farming
```

### Step 2: Configure Connection Settings

Edit these variables in the DAG file according to your environment:

```python
# HDFS Configuration
HDFS_INPUT_PATH = "hdfs://namenode:9000/user/smart_farming_data/*"

# Checkpoint File Location
CHECKPOINT_FILE = "/tmp/last_processed_timestamp.txt"

# MySQL Configuration
MYSQL_URL = "jdbc:mysql://mysql:3306/farm_dwh"
MYSQL_PROPERTIES = {
    "user": "root",
    "password": "root",
    "driver": "com.mysql.cj.jdbc.Driver"
}
```

### Step 3: Create MySQL Database

```sql
CREATE DATABASE IF NOT EXISTS farm_dwh;
USE farm_dwh;
```

The DAG will automatically create the tables on first run.

### Step 4: Initialize Checkpoint (Optional)

For the first run, you can either:

**Option A**: Let it process all historical data
- Delete or don't create the checkpoint file
- DAG will process all available data

**Option B**: Start from a specific timestamp
```bash
echo "2024-01-01 00:00:00" > /tmp/last_processed_timestamp.txt
```

### Step 5: Enable and Trigger the DAG

```bash
# Enable the DAG
airflow dags unpause smart_farming_etl_incremental

# Trigger manual run (optional)
airflow dags trigger smart_farming_etl_incremental
```

## 📊 Data Flow

```
┌─────────────────┐
│   HDFS Source   │
│  (Parquet Files)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Check New Data │
│  (Checkpoint)   │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐  ┌─────────┐
│ Skip  │  │ Extract │
│       │  │Transform│
└───────┘  └────┬────┘
                │
                ▼
         ┌──────────────┐
         │Create Dims & │
         │    Facts     │
         └──────┬───────┘
                │
                ▼
         ┌─────────────┐
         │  Load to    │
         │   MySQL     │
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
         │   Update    │
         │ Checkpoint  │
         └─────────────┘
```

## 🗂️ Output Tables

### Dimension Tables
1. **dim_farm**: Farm information (farm_id, region)
2. **dim_crop**: Crop types
3. **dim_time**: Time dimension (date, year, month, day, week, hour)

### Fact Table
4. **fact_sensor_data**: Main sensor readings with all metrics

### Analytical Tables
5. **moisture_trend**: Soil moisture trends by farm and region
6. **rain_moisture**: Rainfall vs soil moisture correlation
7. **climate_effect**: Climate impact on soil moisture
8. **ph_trend**: Soil pH trends by crop and region
9. **rain_pesticide**: Rainfall vs pesticide usage
10. **sunlight_daily**: Daily sunlight exposure by region
11. **pesticide_trend**: Pesticide usage trends by crop

## ⚙️ Configuration Options

### Schedule Interval

Modify the `schedule_interval` parameter in the DAG:

```python
# Hourly (default)
schedule_interval='@hourly'

# Daily at midnight
schedule_interval='0 0 * * *'

# Every 6 hours
schedule_interval='0 */6 * * *'

# Custom cron expression
schedule_interval='*/30 * * * *'  # Every 30 minutes
```

### Retry Configuration

Adjust retry settings in `default_args`:

```python
default_args = {
    'retries': 3,                    # Number of retries
    'retry_delay': timedelta(minutes=10),  # Delay between retries
}
```

### Email Notifications

Enable email alerts:

```python
default_args = {
    'email': ['your-email@example.com'],
    'email_on_failure': True,
    'email_on_retry': True,
}
```

## 🔍 Monitoring & Troubleshooting

### Check DAG Status
```bash
# View DAG runs
airflow dags list-runs -d smart_farming_etl_incremental

# View task instances
airflow tasks list smart_farming_etl_incremental
```

### View Logs
```bash
# Airflow UI
http://localhost:8080

# Command line
airflow tasks logs smart_farming_etl_incremental <task_id> <execution_date>
```

### Check Checkpoint
```bash
# View current checkpoint
cat /tmp/last_processed_timestamp.txt

# Reset checkpoint (processes all data on next run)
rm /tmp/last_processed_timestamp.txt
```

### Common Issues

#### 1. **No new data being processed**
- Check if checkpoint file exists and timestamp is correct
- Verify HDFS path has new files with timestamps > checkpoint
- Review logs in task `check_new_data`

#### 2. **JDBC Connection Failed**
```bash
# Verify MySQL JDBC driver is in Spark jars
ls $SPARK_HOME/jars/ | grep mysql

# Test MySQL connection
mysql -h mysql -u root -p -e "SHOW DATABASES;"
```

#### 3. **OutOfMemory Errors**
Increase Spark memory in DAG:
```python
spark = SparkSession.builder \
    .appName("ETL_SmartFarming") \
    .config("spark.driver.memory", "4g") \
    .config("spark.executor.memory", "4g") \
    .getOrCreate()
```

#### 4. **Checkpoint Not Updating**
- Ensure all tasks complete successfully
- Check write permissions on `/tmp/last_processed_timestamp.txt`
- Review logs in task `update_checkpoint`

## 📈 Performance Optimization

### 1. Partition Data in HDFS
```python
# Partition by date for faster filtering
df.write.partitionBy("date").parquet(hdfs_path)
```

### 2. Adjust Spark Parallelism
```python
.config("spark.default.parallelism", "100") \
.config("spark.sql.shuffle.partitions", "100")
```

### 3. Use Broadcast Joins
For small dimension tables:
```python
from pyspark.sql.functions import broadcast
df.join(broadcast(dim_table), "key")
```

## 🔐 Security Considerations

### 1. Secure Credentials
Use Airflow Connections or environment variables:

```python
from airflow.hooks.base import BaseHook

# Get MySQL connection from Airflow
mysql_conn = BaseHook.get_connection('mysql_farm_dwh')
MYSQL_PROPERTIES = {
    "user": mysql_conn.login,
    "password": mysql_conn.password,
    "driver": "com.mysql.cj.jdbc.Driver"
}
```

### 2. Checkpoint File Permissions
```bash
chmod 600 /tmp/last_processed_timestamp.txt
chown airflow:airflow /tmp/last_processed_timestamp.txt
```

## 🧪 Testing

### Test Individual Tasks
```bash
# Test check_new_data task
airflow tasks test smart_farming_etl_incremental check_new_data 2024-01-01

# Test extract_transform task
airflow tasks test smart_farming_etl_incremental extract_transform 2024-01-01
```

### Dry Run
```bash
# Run DAG without actually executing tasks
airflow dags test smart_farming_etl_incremental 2024-01-01
```

## 📞 Support & Maintenance

### Daily Checks
- Monitor DAG success rate in Airflow UI
- Verify checkpoint file is updating
- Check MySQL table row counts

### Weekly Maintenance
- Review and clean temporary directories
- Analyze query performance in MySQL
- Archive old logs

### Monthly Tasks
- Review data quality metrics
- Optimize slow-running transformations
- Update outlier detection thresholds if needed

## 📝 Data Processing Details

### Missing Value Handling
- Numeric columns filled with mean values
- Columns: soil_moisture, soil_pH, temperature, humidity, sunlight_intensity

### Outlier Detection
- Uses IQR (Interquartile Range) method
- Removes values outside [Q1 - 1.5*IQR, Q3 + 1.5*IQR]
- Applied to all sensor measurements

### Data Transformations
1. Farm ID extraction from string to integer
2. Timestamp parsing and date part extraction
3. Text column whitespace trimming
4. Type casting to appropriate data types



## 📚 Additional Resources

- [Apache Airflow Documentation](https://airflow.apache.org/docs/)
- [PySpark SQL Documentation](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql.html)
- [MySQL JDBC Driver](https://dev.mysql.com/downloads/connector/j/)

***



# Smart Farming Data Generator

## Overview

This Jupyter notebook generates realistic synthetic smart farming sensor data for an entire year (365 days) at minute-level granularity. The generator simulates environmental conditions and agricultural metrics across 10 different farms in Egypt, producing over 5.2 million data records that model real-world farming scenarios with correlated sensor readings.

## Features

- **Minute-Level Data Generation**: Creates one data point per minute for each farm (525,600 minutes/year)
- **Multi-Farm Simulation**: Generates data for 10 farms across 3 regions in Egypt
- **Realistic Physics-Based Modeling**: Implements correlations between temperature, humidity, sunlight, and other environmental factors
- **Seasonal Variations**: Includes realistic seasonal temperature shifts throughout the year
- **Dynamic Environmental Events**: Simulates rainfall, irrigation, and pesticide application
- **State-Based Generation**: Maintains farm state between readings for realistic temporal continuity
- **Data Quality Features**: Introduces realistic null values (3% probability) to simulate sensor failures
- **CSV Export**: Outputs data in a structured CSV format ready for analysis

## Generated Dataset Statistics

- **Total Records**: 5,256,000 (10 farms × 525,600 minutes)
- **Time Range**: January 1, 2024 to January 1, 2025
- **Temporal Granularity**: 1 minute
- **Number of Farms**: 10
- **Number of Regions**: 3 (Nile Delta, Upper Egypt, Sinai)
- **File Size**: ~500-600 MB (CSV format)

## Farm Configuration

The generator simulates 10 farms across Egypt with different crop types:

### Nile Delta Region
| Farm ID | Crop Type |
|---------|-----------|
| farm_1  | Wheat     |
| farm_2  | Rice      |
| farm_3  | Onion     |

### Upper Egypt Region
| Farm ID | Crop Type |
|---------|-----------|
| farm_4  | Tomato    |
| farm_5  | Dates     |
| farm_6  | Peanuts   |

### Sinai Region
| Farm ID | Crop Type |
|---------|-----------|
| farm_7  | Corn      |
| farm_8  | Olive     |
| farm_9  | Barley    |
| farm_10 | Potato    |

## Data Schema

Each generated record contains the following fields:

| Field | Type | Description | Range/Format |
|-------|------|-------------|--------------|
| `sensor_id` | string (UUID) | Unique identifier for each reading | UUID v5 format |
| `timestamp` | string (ISO 8601) | Recording timestamp | YYYY-MM-DDTHH:MM:SS |
| `soil_moisture` | float | Soil moisture percentage | 0-100% |
| `soil_pH` | float | Soil pH level | 5.0-8.0 |
| `temperature` | float | Temperature in Celsius | 15-35°C |
| `rainfall` | float | Rainfall in mm | 0-80 mm |
| `humidity` | float | Relative humidity percentage | 10-100% |
| `sunlight_intensity` | float | Solar radiation intensity | 0-12 units |
| `pesticide_usage_ml` | float | Pesticide application volume | 0-20 ml |
| `farm_id` | string | Farm identifier | farm_1 to farm_10 |
| `region` | string | Geographic region | NileDelta, UpperEgypt, Sinai |
| `crop_type` | string | Crop being grown | Various (see table above) |

## Data Generation Algorithm

### 1. State Initialization
Each farm starts with randomized initial conditions:
- Soil moisture: 30-45%
- Soil pH: 6.0-7.0
- Temperature: 20-30°C

### 2. Time-Based Calculations

#### Sunlight Intensity
- **Daytime (6 AM - 6 PM)**: Bell curve distribution peaking at noon
- **Nighttime**: Zero sunlight
- **Seasonal adjustment**: ±20% variation based on day of year
- **Formula**: `12 × exp(-((hour - 12)² / 18)) × seasonal_factor`

#### Temperature
- **Base calculation**: Influenced by sunlight intensity
- **Seasonal variation**: ±5°C sine wave over the year
- **Daily fluctuation**: Random variations of ±0.3°C per minute
- **Inertia**: Gradual temperature changes (30% adjustment rate)

#### Rainfall
- **Probability**: 1.4% chance per minute (~7,300 events/year)
- **Amount**: Random 10-80 mm when occurring
- **Impact**: Affects soil moisture and pH

### 3. Interdependent Variables

#### Soil Moisture
- **Evaporation**: Decreases based on temperature
- **Rainfall absorption**: Increases by 60% of rainfall amount
- **Auto-irrigation**: Triggered when moisture drops below 20%
- **Bounds**: Clamped to 0-100%

#### Humidity
- **Temperature correlation**: Inverse relationship (−0.8 × temperature deviation)
- **Rainfall impact**: Increases with precipitation (+0.25 × rainfall)
- **Random variation**: ±3% per reading
- **Bounds**: 10-100%

#### Soil pH
- **Drift**: Random changes of ±0.02 per minute
- **Rainfall acidification**: Decreases with heavy rain
- **Moisture alkalinization**: Increases when soil is very moist
- **Pesticide impact**: Decreases by 0.2 when pesticides applied

#### Pesticide Usage
- **Post-rainfall application**: 70% probability after rain events
- **Scheduled application**: Weekly intervals
- **Amount**: Random 5-20 ml per application

### 4. Stability Mechanisms
To prevent unrealistic outliers, the system includes mean-reversion:
- Temperature extremes (±15°C from 25°C midpoint)
- Soil moisture extremes (<10% or >90%)
- Humidity extremes (<15% or >90%)
- pH extremes (<5.0 or >8.0)

### 5. Data Quality Simulation
- **Missing values**: 3% probability of null for sensor readings
- **Affected fields**: soil_moisture, soil_pH, temperature, humidity, sunlight_intensity

## Prerequisites

### Required Python Packages
```python
import json
import uuid
import random
import math
import csv
from datetime import datetime, timedelta
```

All packages are part of Python's standard library - no external dependencies required!

## Usage

### Step 1: Open the Notebook
```bash
jupyter notebook generate_data.ipynb
```

### Step 2: Run All Cells
Execute the main data generation cell. The process will:
1. Initialize 10 farm states
2. Generate data for each minute of the year
3. Display progress every 30 days (43,200 minutes)
4. Save output to `smart_farming_data.csv`

### Step 3: Monitor Progress
Expected output:
```
✅ Generated 43200/525600 minutes (8.2%)
✅ Generated 86400/525600 minutes (16.4%)
✅ Generated 129600/525600 minutes (24.7%)
...
✅ Generated 518400/525600 minutes (98.6%)
🚀 Successfully generated smart_farming_data.csv with 5,256,000 records!
📊 Time range: Jan 1, 2024 to Jan 1, 2025
```

## Configuration Options

### Modify Time Range
```python
# Current: 1 year at minute granularity
total_minutes = 60 * 24 * 365

# Example: 6 months
total_minutes = 60 * 24 * 183

# Example: 1 month at 5-minute intervals
total_minutes = 12 * 24 * 30
# Then modify: timedelta(minutes=5)
```

### Adjust Farm Configuration
```python
# Add more farms
farms.append({
    "farm_id": "farm_11", 
    "region": "Alexandria", 
    "crop_type": "Cotton"
})

# Modify regions
farms[0]["region"] = "NewRegionName"
```

### Customize Environmental Parameters
```python
# Modify midpoints for mean reversion
midpoints = {
    "soil_moisture": 45,  # Change from 40
    "temperature": 28,     # Change from 25
    "humidity": 60,        # Change from 55
    "soil_pH": 6.8        # Change from 6.5
}

# Adjust rainfall probability
if random.random() < 0.02:  # Change from 0.014 (2% instead of 1.4%)
    rainfall = random.uniform(10, 80)
```

### Modify Null Value Probability
```python
# Current: 3% probability
if random.random() < 0.05:  # Change to 5%
    record[key] = None
```

## Performance Considerations

### Generation Time
- **10 farms, 1 year**: ~15-25 minutes (depending on hardware)
- **Memory usage**: ~2-3 GB during generation
- **Output file**: ~500-600 MB

### Optimization Tips

1. **Reduce temporal granularity**:
   ```python
   # 5-minute intervals instead of 1-minute
   for minute in range(0, total_minutes, 5):
   ```

2. **Reduce number of farms**:
   ```python
   farms = farms[:5]  # Use only first 5 farms
   ```

3. **Shorter time period**:
   ```python
   total_minutes = 60 * 24 * 30  # Just one month
   ```

## Output File Format

### CSV Structure
```csv
sensor_id,timestamp,soil_moisture,soil_pH,temperature,rainfall,humidity,sunlight_intensity,pesticide_usage_ml,farm_id,region,crop_type
550e8400-e29b-41d4-a716-446655440000,2024-01-01T00:01:00,42.5,6.8,22.3,0,52.1,0,0,farm_1,NileDelta,Wheat
...
```

### Example Record
```json
{
  "sensor_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2024-01-01T00:01:00",
  "soil_moisture": 42.5,
  "soil_pH": 6.8,
  "temperature": 22.3,
  "rainfall": 0,
  "humidity": 52.1,
  "sunlight_intensity": 0,
  "pesticide_usage_ml": 0,
  "farm_id": "farm_1",
  "region": "NileDelta",
  "crop_type": "Wheat"
}
```

## Use Cases

This synthetic dataset is ideal for:

1. **Machine Learning Training**: Develop crop yield prediction models
2. **Time Series Analysis**: Study seasonal patterns and trends
3. **Anomaly Detection**: Identify unusual environmental conditions
4. **IoT System Testing**: Simulate real-time sensor data streams
5. **Dashboard Development**: Create agricultural monitoring visualizations
6. **ETL Pipeline Testing**: Validate data processing workflows
7. **Database Performance Testing**: Benchmark large-scale data ingestion

## Data Quality & Realism

### Realistic Features
- ✅ Correlated environmental variables
- ✅ Seasonal temperature variations
- ✅ Diurnal (day/night) cycles
- ✅ Realistic rainfall patterns
- ✅ Temporal continuity (state-based)
- ✅ Sensor failure simulation (nulls)
- ✅ Deterministic UUIDs (reproducible)

### Limitations
- ⚠️ Simplified weather patterns
- ⚠️ No geographic variation between regions
- ⚠️ No crop-specific growth modeling
- ⚠️ Basic rainfall probability model
- ⚠️ No extreme weather events (droughts, floods)

## Troubleshooting

### Memory Issues
```python
# Write in batches instead of keeping all in memory
# Current implementation already uses streaming to CSV
```

### Slow Generation
- Reduce time range or number of farms
- Increase time interval between readings
- Use PyPy instead of CPython for ~2x speedup

### File Not Created
```python
# Check permissions in current directory
import os
print(os.getcwd())
print(os.access('.', os.W_OK))  # Should return True
```

## Reproducibility

The generator uses a fixed random seed:
```python
random.seed(42)
```

Running the notebook multiple times will produce **identical data**. To generate different datasets, change the seed:
```python
random.seed(12345)  # Different seed = different data
```



## Related Scripts

- `kafka_producer_from_csv.ipynb` - Stream this data to Kafka
- `ETL_SmartFarming.ipynb` - Process and transform the data
- `spark_code.ipynb` - Analyze with Apache Spark

***


# Kafka Producer from CSV - Smart Farming Data

## Overview

This Jupyter notebook implements a Kafka producer that reads smart farming sensor data from a CSV file and streams it to a Kafka topic. The producer is designed to simulate real-time data ingestion by sending records with a small delay between each message.

## Features

- **CSV Data Ingestion**: Reads farming sensor data from a CSV file
- **JSON Serialization**: Automatically serializes Python dictionaries to JSON format
- **Data Type Conversion**: Intelligently converts numeric fields and handles null values
- **Progress Tracking**: Displays real-time progress updates every 1,000 records
- **Rate Limiting**: Includes configurable delay between records (1ms default)
- **Robust Data Handling**: Properly handles missing values and type conversions

## Prerequisites

### Required Software
- Python 3.x
- Jupyter Notebook/JupyterLab
- Apache Kafka cluster (running and accessible)

### Required Python Packages
```bash
pip install kafka-python
```

### Kafka Setup
- Kafka broker must be running and accessible at `broker:29092`
- Ensure the Kafka topic exists or auto-creation is enabled

## Data Schema

The script expects a CSV file with the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `soil_moisture` | float | Soil moisture percentage |
| `soil_pH` | float | Soil pH level |
| `temperature` | float | Temperature in degrees |
| `rainfall` | float | Rainfall measurement |
| `humidity` | float | Humidity percentage |
| `sunlight_intensity` | float | Sunlight intensity measurement |
| `pesticide_usage_ml` | float | Pesticide usage in milliliters |
| Other fields | string | Additional categorical or text data |

**Note**: Numeric fields are automatically converted to float. Empty values or 'None' strings are converted to Python `None`.

## Configuration

### File Paths
```python
csv_filename = '/home/jovyan/smart_farming_data.csv'
```
Update this path to match your CSV file location.

### Kafka Connection
```python
bootstrap_servers='broker:29092'
```
Modify the bootstrap servers to match your Kafka cluster configuration.

### Kafka Topic
```python
topic = 'smart_farming_data'
```
Change the topic name as needed for your use case.

### Streaming Rate
```python
time.sleep(0.001)  # 1ms delay between records
```
Adjust this value to control the rate of data streaming:
- `0.001` = 1ms (up to 1,000 records/second)
- `0.01` = 10ms (up to 100 records/second)
- `0.1` = 100ms (up to 10 records/second)

## Usage

### Step 1: Prepare Your Environment
Ensure your Kafka cluster is running and accessible:
```bash
# Check if Kafka is running
docker ps | grep kafka
```

### Step 2: Verify CSV File
Ensure your CSV file exists at the specified path:
```bash
ls -lh /home/jovyan/smart_farming_data.csv
```

### Step 3: Run the Notebook
1. Open the notebook in Jupyter
2. Run all cells or execute the main cell
3. Monitor the progress output

### Expected Output
```
📖 Reading data from /home/jovyan/smart_farming_data.csv...
✅ Sent 1,000 records to Kafka...
✅ Sent 2,000 records to Kafka...
✅ Sent 3,000 records to Kafka...
...
🚀 Successfully sent 140,000 records to Kafka topic 'smart_farming_data'
```

## Data Processing Details

### Type Conversion Logic

The script implements intelligent type conversion:

1. **Empty Values**: Empty strings or 'None' strings → Python `None`
2. **Numeric Fields**: Converted to `float` for sensor measurements
3. **String Fields**: Preserved as strings for categorical data
4. **Error Handling**: Failed conversions default to `None`

### Example Transformation

**Input CSV Row:**
```csv
soil_moisture,soil_pH,temperature,crop_type
45.5,6.8,25.3,Wheat
```

**Output JSON:**
```json
{
  "soil_moisture": 45.5,
  "soil_pH": 6.8,
  "temperature": 25.3,
  "crop_type": "Wheat"
}
```

## Performance Considerations

### Current Configuration
- **Record Rate**: ~1,000 records/second (with 1ms delay)
- **140,000 Records**: ~2-3 minutes processing time
- **Memory Usage**: Minimal (streaming approach)

### Optimization Tips

1. **Increase Throughput**: Reduce or remove `time.sleep()` delay
2. **Batch Sending**: Use `batch_size` parameter in KafkaProducer
3. **Compression**: Enable compression for large payloads
   ```python
   producer = KafkaProducer(
       compression_type='gzip'
   )
   ```
4. **Async Sending**: Remove `producer.flush()` calls for async operation

## Troubleshooting

### Common Issues

#### 1. Connection Refused
```
NoBrokersAvailable: NoBrokersAvailable
```
**Solution**: Verify Kafka broker is running and hostname/port are correct

#### 2. Topic Not Found
```
UnknownTopicOrPartitionError
```
**Solution**: Enable auto-create topics in Kafka or create topic manually:
```bash
kafka-topics --create --topic smart_farming_data \
  --bootstrap-server localhost:9092 \
  --partitions 3 --replication-factor 1
```

#### 3. CSV File Not Found
```
FileNotFoundError: [Errno 2] No such file or directory
```
**Solution**: Verify the CSV file path is correct and file exists

#### 4. Encoding Issues
**Solution**: The script uses UTF-8 encoding. If you have special characters, verify your CSV is UTF-8 encoded

### Debug Mode

Enable verbose logging:
```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

## Advanced Configuration

### Custom Serialization
```python
def custom_serializer(value):
    # Add custom logic here
    return json.dumps(value, default=str).encode('utf-8')

producer = KafkaProducer(
    value_serializer=custom_serializer
)
```

### Partitioning Strategy
```python
producer = KafkaProducer(
    partitioner=lambda key, all_partitions, available: hash(key) % len(all_partitions)
)

# Send with key for partitioning
producer.send(topic, key=b'field_id', value=record)
```

### Error Callbacks
```python
def on_send_success(record_metadata):
    print(f"Sent to {record_metadata.topic}")

def on_send_error(excp):
    print(f"Error: {excp}")

producer.send(topic, value=record).add_callback(on_send_success).add_errback(on_send_error)
```

## Monitoring

### Kafka Consumer Test
Verify messages are being produced:
```bash
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic smart_farming_data --from-beginning
```

### Check Topic Details
```bash
kafka-topics --describe --topic smart_farming_data \
  --bootstrap-server localhost:9092
```

## Best Practices

1. **Always call `producer.flush()`**: Ensures all messages are sent before exit
2. **Handle exceptions**: Wrap producer code in try-except blocks
3. **Close connections**: Use context managers or explicitly close producer
4. **Monitor lag**: Check consumer lag if processing real-time data
5. **Data validation**: Validate data before sending to Kafka
