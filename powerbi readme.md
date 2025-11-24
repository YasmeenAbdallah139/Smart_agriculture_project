# FarmDWH – Agriculture Analytics Dashboard (Power BI)
## Overview
![Dashboard Preview](images/power_bi1.png "FarmDWH Dashboard Overview")
## Soil Health
![Dashboard Preview](images/power_bi2.png "FarmDWH Dashboard Overview")
## Pesticide & Chemical Usage
![Dashboard Preview](images/power_bi3.png "FarmDWH Dashboard Overview")
## Climate Trend
![Dashboard Preview](images/power_bi4.png "FarmDWH Dashboard Overview")


## Overview of the dashboard 
This Power BI dashboard provides real-time insights into farm operations, soil health, pesticide & chemical usage, moisture levels, and climate trends for farms in Egypt (Nile Delta, Sinai, Upper Egypt, and surrounding regions).

The data is sourced from the **farm_dwh** data warehouse (SQL Server / Azure SQL / MySQL – adjust according to your environment).

## Dashboard Pages

| Page                  | Purpose                                                                 |
|-----------------------|-------------------------------------------------------------------------|
| **Overview**          | High-level KPIs: Total pesticide usage (57.88K ml), Avg temperature (24.09°C), Avg humidity (51.79%), Rainfall count (2,644 events) |
| **Soil Health**       | Soil moisture trends, sunlight intensity, temperature extremes per farm |
| **Pesticide & Chemical Usage** | Pesticide consumption by farm, week, crop type, and region. Tracks total usage (ml) and identifies high-usage periods |
| **Climate Trends**    | Weekly/monthly temperature, humidity, and rainfall patterns with min/max/avg gauges |

## Key Metrics & Visuals
- Total Pesticide Usage: **57.88K ml**
- Average Temperature: **24.09°C** (Max 73.8°C | Min 17.47°C)
- Average Humidity: **51.79%** (Max 128.6% | Min 39.4%)
- Rainfall Events: **2,644**
- Soil Moisture (avg): **6.87K**
- Sunlight Intensity (max): **661.7K**

## Data Source
- **Database**: `farm_dwh` (SQL-based data warehouse)
- **Main Fact Tables** (example – update with your actual tables):
  - `fact_pesticide_usage`
  - `fact_weather_daily`
  - `fact_soil_moisture`
  - `fact_crop_yield` (if available)
- **Dimension Tables**:
  - `dim_farm`, `dim_crop`, `dim_date`, `dim_region`

### Connection Details (Power BI)
- Mode: **Import** or **DirectQuery** (currently using Import for performance)
- Gateway: Enterprise gateway (if on-prem SQL) or built-in for cloud

## How to Open & Refresh
1. Open `FarmDWH_Dashboard.pbix` in Power BI Desktop (latest version recommended)
2. Go to **Home → Transform data → Data source settings**
3. Update/clear permissions for the `farm_dwh` server if needed
4. Click **Refresh** to pull the latest data
5. Publish to Power BI Service (if you have a workspace)

## Filters & Slicers Available
- Region (Nile Delta, Sinai, Upper Egypt, etc.)
- Crop Type (Wheat, Rice, Tomato, Potato, Onion, Corn, Dates, Olive, Barley, Peanuts)
- Farm ID
- Date / Week / Month

