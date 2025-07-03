# 🏗️ Architecture Documentation

## Overview

This document provides a detailed technical overview of the Crypto Price Prediction Pipeline architecture, including the Medallion data architecture, Azure Synapse components, and machine learning pipeline design.

---

## 🏛️ Medallion Architecture

### Bronze Layer (Raw Data)

**Purpose**: Store raw, unprocessed data from external sources
**Location**: `abfss://bronze@lakeiqbetim.dfs.core.windows.net/`

#### Data Source
- **API**: CoinGecko REST API
- **Endpoint**: `https://api.coingecko.com/api/v3/coins/markets`
- **Parameters**: `vs_currency=usd&order=market_cap_desc&per_page=250&page=1`
- **Format**: JSON response

#### Storage Details
```json
{
  "container": "bronze",
  "path": "coins/raw/",
  "format": "JSON",
  "partitioning": "date-based",
  "retention": "30 days"
}
```

#### Data Schema (Raw)
```json
{
  "id": "bitcoin",
  "symbol": "btc",
  "name": "Bitcoin",
  "current_price": 43250.0,
  "market_cap": 847234567890,
  "market_cap_rank": 1,
  "total_volume": 23456789012,
  "high_24h": 44500.0,
  "low_24h": 42000.0,
  "price_change_24h": 1250.0,
  "price_change_percentage_24h": 2.98,
  "circulating_supply": 19567890.0,
  "total_supply": 21000000.0,
  "ath": 69000.0,
  "ath_change_percentage": -37.32,
  "roi": {
    "percentage": 1250.5,
    "currency": "usd"
  }
}
```

---

### Silver Layer (Cleaned & Transformed)

**Purpose**: Store cleaned, validated, and feature-engineered data
**Location**: `abfss://silver@lakeiqbetim.dfs.core.windows.net/`

#### Processing Pipeline
- **Technology**: PySpark
- **Notebook**: `data_transformation_silver.json`
- **Execution**: Azure Synapse Spark Pool

#### Feature Engineering
```python
# Calculated Features
df = df.withColumn("price_volatility", 
                   col("high_24h") - col("low_24h"))

df = df.withColumn("ath_pct", 
                   (col("current_price") - col("ath")) / col("ath") * 100)

df = df.withColumn("roi_percentage", 
                   col("roi.percentage"))

# Data Quality Checks
df = df.filter(col("current_price").isNotNull())
df = df.filter(col("market_cap") > 0)
```

#### Storage Details
```json
{
  "container": "silver",
  "path": "coins/processed/",
  "format": "Parquet",
  "compression": "snappy",
  "partitioning": "date-based"
}
```

#### Data Schema (Silver)
```json
{
  "id": "bitcoin",
  "symbol": "btc",
  "name": "Bitcoin",
  "current_price": 43250.0,
  "market_cap": 847234567890,
  "total_volume": 23456789012,
  "price_change_percentage_24h": 2.98,
  "circulating_supply": 19567890.0,
  "price_volatility": 2500.0,
  "ath_pct": -37.32,
  "roi_percentage": 1250.5,
  "processed_date": "2024-12-01"
}
```

---

### Gold Layer (ML-Ready)

**Purpose**: Store aggregated, business-ready data for machine learning
**Location**: `abfss://gold@lakeiqbetim.dfs.core.windows.net/`

#### ML Processing
- **Technology**: scikit-learn + XGBoost
- **Notebook**: `ML_CoinGecko.json`
- **Execution**: Azure Synapse Spark Pool

#### Feature Selection
```python
# Features for ML model
features = [
    "market_cap",
    "total_volume", 
    "price_change_percentage_24h",
    "circulating_supply",
    "price_volatility",
    "ath_pct",
    "roi_percentage"
]

# Target variable
target = "current_price"
```

#### Storage Details
```json
{
  "container": "gold",
  "path": "coins/ml_ready/",
  "format": "Parquet",
  "compression": "snappy",
  "metadata": {
    "model_version": "1.0",
    "features": ["market_cap", "total_volume", ...],
    "target": "current_price"
  }
}
```

---

## 🔄 Pipeline Orchestration

### Main Pipeline: `Pipeline 1.json`

```json
{
  "name": "Pipeline 1",
  "activities": [
    {
      "name": "bronze_execute",
      "type": "ExecutePipeline",
      "pipeline": {
        "referenceName": "CoinGecko",
        "type": "PipelineReference"
      }
    },
    {
      "name": "silver_execute", 
      "type": "ExecutePipeline",
      "dependsOn": ["bronze_execute"],
      "pipeline": {
        "referenceName": "coingecko_notebook",
        "type": "PipelineReference"
      }
    },
    {
      "name": "gold_execute",
      "type": "ExecutePipeline", 
      "dependsOn": ["silver_execute"],
      "pipeline": {
        "referenceName": "ML_coinGecko",
        "type": "PipelineReference"
      }
    }
  ]
}
```

### Pipeline Dependencies
```
Bronze Layer (CoinGecko.json)
    ↓
Silver Layer (coingecko_notebook.json)  
    ↓
Gold Layer (ML_coinGecko.json)
```

---

## 🧠 Machine Learning Architecture

### Model Pipeline

#### 1. Data Preparation
```python
# Load data from silver layer
df = pd.read_parquet("abfss://silver@lakeiqbetim.dfs.core.windows.net/coins/ml_ready/")

# Feature selection
X = df.drop(columns=["current_price", "id", "symbol"])
y = df["current_price"]

# Train/test split
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
```

#### 2. Model Training
```python
# XGBoost with GridSearchCV
param_grid = {
    "max_depth": [3, 5, 7],
    "learning_rate": [0.05, 0.1],
    "n_estimators": [100, 200],
    "subsample": [0.8, 1.0]
}

grid = GridSearchCV(XGBRegressor(random_state=42), param_grid, cv=3, scoring="r2")
grid.fit(X_train, y_train)
```

#### 3. Model Evaluation
```python
# Metrics calculation
y_pred = grid.best_estimator_.predict(X_test)

mae = mean_absolute_error(y_test, y_pred)
rmse = mean_squared_error(y_test, y_pred, squared=False)
r2 = r2_score(y_test, y_pred)
```

### Model Performance

#### Current Results
| Metric | Value | Interpretation |
|--------|-------|----------------|
| MAE | 5,554 USD | Average absolute prediction error |
| RMSE | 23,923 USD | Root mean square error |
| R² | 0.46 | 46% of variance explained |

#### Feature Importance
1. **market_cap** (0.35) - Market capitalization
2. **total_volume** (0.28) - Trading volume
3. **circulating_supply** (0.18) - Available supply
4. **price_change_percentage_24h** (0.12) - Price momentum
5. **roi_percentage** (0.07) - Return on investment

---

## 🔧 Azure Services Configuration

### Synapse Workspace
- **Name**: alex2026
- **Region**: East US
- **Resource Group**: new_recurse_service

### Spark Pool Configuration
```json
{
  "name": "smalsize",
  "nodeCount": 10,
  "cores": 4,
  "memory": 28,
  "sparkVersion": "3.4",
  "automaticScaleJobs": false
}
```

### Linked Services
1. **AzureDataLakeStorage1**: Data Lake Gen2 connection
2. **HttpServer1**: CoinGecko API connection
3. **AzureMLService1**: Azure ML integration

### Triggers
- **raw_data.json**: Scheduled trigger for data ingestion
- **Frequency**: Daily at 00:00 UTC

---

## 📊 Data Flow Diagram

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  CoinGecko API  │    │  Azure Synapse  │    │  Azure Data     │
│                 │    │  Pipeline       │    │  Lake Gen2      │
│ • REST API      │───▶│ • Orchestration │───▶│ • Bronze Layer  │
│ • JSON Response │    │ • Transformation│    │ • Silver Layer  │
│ • Real-time     │    │ • ML Processing │    │ • Gold Layer    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │  ML Model       │
                       │                 │
                       │ • XGBoost       │
                       │ • Predictions   │
                       │ • Metrics       │
                       └─────────────────┘
```

---

## 🔒 Security & Compliance

### Data Security
- **Encryption**: Azure Data Lake Gen2 encryption at rest
- **Authentication**: Azure AD integration
- **Authorization**: Role-based access control (RBAC)

### API Security
- **Rate Limiting**: CoinGecko API rate limits
- **Authentication**: API key management (if required)
- **Monitoring**: API call monitoring and alerting

### Compliance
- **Data Retention**: 30-day retention policy
- **Audit Logging**: Azure Monitor integration
- **GDPR**: Data privacy compliance

---

## 📈 Scalability Considerations

### Horizontal Scaling
- **Spark Pool**: Auto-scaling based on workload
- **Data Lake**: Unlimited storage capacity
- **Pipelines**: Parallel execution capabilities

### Performance Optimization
- **Parquet Format**: Columnar storage for fast queries
- **Partitioning**: Date-based partitioning for efficient filtering
- **Caching**: Spark DataFrame caching for repeated operations

### Cost Optimization
- **Compute**: Pay-per-use Spark pools
- **Storage**: Tiered storage (Hot/Cool/Archive)
- **Monitoring**: Cost tracking and optimization recommendations

---

## 🚨 Monitoring & Alerting

### Pipeline Monitoring
- **Success/Failure**: Pipeline execution status
- **Duration**: Execution time tracking
- **Data Quality**: Record counts and validation

### Model Monitoring
- **Performance**: MAE, RMSE, R² tracking
- **Drift**: Feature distribution changes
- **Predictions**: Prediction accuracy over time

### Alerting
- **Pipeline Failures**: Email/SMS notifications
- **Model Degradation**: Performance threshold alerts
- **Data Quality**: Anomaly detection alerts

---

*Last updated: December 2024* 