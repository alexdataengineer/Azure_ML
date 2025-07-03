# 📁 Project Structure Documentation

## Overview

This document provides a comprehensive overview of the project structure, explaining the purpose and organization of each directory and file in the Crypto Price Prediction Pipeline.

---

## 🏗️ Directory Structure

```
Azure_ML/
├── 📁 .git/                          # Git version control
├── 📁 docs/                          # Documentation files
├── 📁 pipeline/                      # Azure Synapse pipeline definitions
├── 📁 notebook/                      # Jupyter notebooks for ML
├── 📁 dataset/                       # Dataset configurations
├── 📁 linkedService/                 # Azure service connections
├── 📁 trigger/                       # Pipeline triggers
├── 📁 sqlscript/                     # SQL scripts (if any)
├── 📁 integrationRuntime/            # Integration runtime configs
├── 📁 credential/                    # Credential management
├── 📄 README.md                      # Main project documentation
├── 📄 requirements.txt               # Python dependencies
└── 📄 publish_config.json           # Azure publish configuration
```

---

## 📚 Documentation (`docs/`)

### Purpose
Contains all project documentation, guides, and technical specifications.

### Files
- **`ARCHITECTURE.md`**: Detailed technical architecture documentation
- **`SETUP.md`**: Step-by-step setup and deployment guide
- **`MODEL_PERFORMANCE.md`**: ML model performance analysis
- **`PROJECT_STRUCTURE.md`**: This file - project organization guide

### Usage
```bash
# View documentation
cat docs/README.md
cat docs/ARCHITECTURE.md
cat docs/SETUP.md
```

---

## 🔄 Pipelines (`pipeline/`)

### Purpose
Contains Azure Synapse pipeline definitions in JSON format.

### Files

#### `Pipeline 1.json` - Main Orchestration Pipeline
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

**Purpose**: Orchestrates the entire data pipeline flow from bronze to gold layer.

#### `CoinGecko.json` - Bronze Layer Pipeline
**Purpose**: Extracts data from CoinGecko API and stores in bronze layer.

**Key Components**:
- **Source**: CoinGecko REST API
- **Sink**: Azure Data Lake Gen2 (bronze container)
- **Format**: JSON
- **Frequency**: Triggered

#### `coingecko_notebook.json` - Silver Layer Pipeline
**Purpose**: Processes raw data through PySpark notebook.

**Key Components**:
- **Activity**: Synapse Notebook
- **Notebook**: `data_transformation_silver.json`
- **Spark Pool**: `smalsize`
- **Output**: Parquet format in silver layer

#### `ML_coinGecko.json` - Gold Layer Pipeline
**Purpose**: Executes machine learning model training.

**Key Components**:
- **Activity**: Synapse Notebook
- **Notebook**: `ML_CoinGecko.json`
- **Spark Pool**: `smalsize`
- **Output**: Trained model and predictions

### Pipeline Dependencies
```
Bronze Layer (CoinGecko.json)
    ↓
Silver Layer (coingecko_notebook.json)
    ↓
Gold Layer (ML_coinGecko.json)
```

---

## 📓 Notebooks (`notebook/`)

### Purpose
Contains Jupyter notebooks for data processing and machine learning.

### Files

#### `ML_CoinGecko.json` - Main ML Training Notebook
**Purpose**: Primary machine learning notebook for model training and evaluation.

**Key Sections**:
```python
# 1. Data Loading
df = pd.read_parquet("abfss://silver@lakeiqbetim.dfs.core.windows.net/coins/ml_ready/")

# 2. Feature Selection
X = df.drop(columns=["current_price", "id", "symbol"])
y = df["current_price"]

# 3. Model Training
model = RandomForestRegressor(random_state=42)
model.fit(X_train, y_train)

# 4. XGBoost with GridSearchCV
grid = GridSearchCV(XGBRegressor(random_state=42), param_grid, cv=3, scoring="r2")

# 5. Model Evaluation
print("MAE:", mean_absolute_error(y_test, y_pred))
print("RMSE:", mean_squared_error(y_test, y_pred, squared=False))
print("R²:", r2_score(y_test, y_pred))

# 6. Feature Importance
importances = best_model.feature_importances_
feature_names = X.columns
```

#### `data_transformation_silver.json` - Data Transformation Notebook
**Purpose**: Transforms bronze layer data into silver layer format.

**Key Operations**:
```python
# Feature Engineering
df = df.withColumn("price_volatility", col("high_24h") - col("low_24h"))
df = df.withColumn("ath_pct", (col("current_price") - col("ath")) / col("ath") * 100)
df = df.withColumn("roi_percentage", col("roi.percentage"))

# Data Quality Checks
df = df.filter(col("current_price").isNotNull())
df = df.filter(col("market_cap") > 0)

# Save to Silver Layer
df.write.mode("overwrite").parquet("abfss://silver@lakeiqbetim.dfs.core.windows.net/coins/processed/")
```

#### `silver_transform.json` - Alternative Silver Processing
**Purpose**: Alternative data transformation approach.

#### `gold_layer.json` - Gold Layer Processing
**Purpose**: Final data preparation for machine learning.

### Notebook Configuration
```json
{
  "bigDataPool": {
    "referenceName": "smalsize",
    "type": "BigDataPoolReference"
  },
  "sessionProperties": {
    "driverMemory": "28g",
    "driverCores": 4,
    "executorMemory": "28g",
    "executorCores": 4,
    "numExecutors": 2
  }
}
```

---

## 📊 Datasets (`dataset/`)

### Purpose
Contains dataset definitions for Azure Synapse pipelines.

### Files

#### `RestResource1.json` - CoinGecko API Dataset
```json
{
  "name": "RestResource1",
  "properties": {
    "linkedServiceName": {
      "referenceName": "HttpServer1",
      "type": "LinkedServiceReference"
    },
    "type": "RestResource",
    "typeProperties": {
      "relativeUrl": "/api/v3/coins/markets",
      "requestMethod": "GET",
      "additionalHeaders": {
        "Accept": "application/json"
      }
    }
  }
}
```

**Purpose**: Defines the CoinGecko API as a data source.

#### `Json1.json`, `Json2.json`, `Json3.json` - JSON Datasets
**Purpose**: Various JSON dataset configurations for different data sources.

#### `spacexfile.json`, `sparcexfile.json` - Additional Datasets
**Purpose**: Additional dataset configurations (may be legacy or for other projects).

### Dataset Types
- **REST API**: External API connections
- **JSON**: JSON file storage
- **Parquet**: Optimized columnar storage
- **CSV**: Comma-separated values

---

## 🔗 Linked Services (`linkedService/`)

### Purpose
Contains Azure service connection configurations.

### Files

#### `AzureDataLakeStorage1.json` - Data Lake Storage
```json
{
  "name": "AzureDataLakeStorage1",
  "properties": {
    "type": "AzureBlobFS",
    "typeProperties": {
      "url": "https://lakeiqbetim.dfs.core.windows.net"
    }
  }
}
```

**Purpose**: Connection to Azure Data Lake Gen2 storage account.

#### `HttpServer1.json` - HTTP Server Connection
```json
{
  "name": "HttpServer1",
  "properties": {
    "type": "HttpServer",
    "typeProperties": {
      "url": "https://api.coingecko.com",
      "authenticationType": "Anonymous"
    }
  }
}
```

**Purpose**: Connection to CoinGecko API.

#### `AzureMLService1.json` - Azure ML Service
```json
{
  "name": "AzureMLService1",
  "properties": {
    "type": "AzureMLService",
    "typeProperties": {
      "subscriptionId": "c13fe431-45c7-45ed-aafb-780b48928c18",
      "resourceGroupName": "new_recurse_service",
      "mlWorkspaceName": "alex2026-ml"
    }
  }
}
```

**Purpose**: Connection to Azure Machine Learning service.

#### Other Linked Services
- **`alex2026-WorkspaceDefaultSqlServer.json`**: Default SQL Server connection
- **`alex2026-WorkspaceDefaultStorage.json`**: Default storage connection
- **`API_work_bank_API.json`**: Additional API connection
- **`Ls_spacex_http.json`**: SpaceX API connection (legacy)

---

## ⚡ Triggers (`trigger/`)

### Purpose
Contains pipeline trigger configurations.

### Files

#### `raw_data.json` - Scheduled Trigger
```json
{
  "name": "raw_data",
  "properties": {
    "type": "ScheduleTrigger",
    "typeProperties": {
      "recurrence": {
        "frequency": "Day",
        "interval": 1,
        "startTime": "2024-12-01T00:00:00Z",
        "timeZone": "UTC"
      }
    }
  }
}
```

**Purpose**: Triggers daily pipeline execution at 00:00 UTC.

### Trigger Types
- **ScheduleTrigger**: Time-based execution
- **TumblingWindowTrigger**: Window-based execution
- **BlobEventsTrigger**: Event-based execution
- **CustomEventsTrigger**: Custom event triggers

---

## 🔧 Configuration Files

### `publish_config.json`
```json
{
  "name": "publish_config",
  "properties": {
    "publishBranch": "main",
    "publishCodeOnly": false
  }
}
```

**Purpose**: Azure Synapse publish configuration for CI/CD.

### `requirements.txt`
**Purpose**: Python package dependencies for the project.

**Key Packages**:
- **pandas**: Data manipulation
- **numpy**: Numerical computing
- **scikit-learn**: Machine learning
- **xgboost**: Gradient boosting
- **pyspark**: Big data processing
- **matplotlib/seaborn**: Visualization

---

## 🔐 Security & Credentials

### `credential/` Directory
**Purpose**: Contains credential management configurations.

**Security Best Practices**:
- Store credentials in Azure Key Vault
- Use managed identities when possible
- Implement least privilege access
- Regular credential rotation

### `integrationRuntime/` Directory
**Purpose**: Contains integration runtime configurations for data movement.

---

## 📋 File Naming Conventions

### Pipeline Files
- **Main orchestration**: `Pipeline 1.json`
- **Layer-specific**: `{LayerName}.json` (e.g., `CoinGecko.json`)
- **Notebook pipelines**: `{purpose}_notebook.json`

### Notebook Files
- **ML training**: `ML_{purpose}.json`
- **Data transformation**: `{layer}_transform.json`
- **Layer processing**: `{layer}_layer.json`

### Dataset Files
- **API sources**: `RestResource{number}.json`
- **Storage formats**: `{format}{number}.json`
- **Specific sources**: `{source}_{format}.json`

---

## 🚀 Deployment Structure

### Development Environment
```
Azure_ML/
├── 📁 dev/                    # Development-specific configs
├── 📁 test/                   # Test configurations
└── 📁 prod/                   # Production configurations
```

### Environment-Specific Files
- **Development**: `config-dev.json`
- **Testing**: `config-test.json`
- **Production**: `config-prod.json`

---

## 📈 Monitoring & Logging

### Log Files Structure
```
logs/
├── 📁 pipeline/               # Pipeline execution logs
├── 📁 notebook/               # Notebook execution logs
├── 📁 model/                  # Model training logs
└── 📁 monitoring/             # Performance monitoring logs
```

### Metrics Collection
- **Pipeline metrics**: Execution time, success/failure rates
- **Model metrics**: MAE, RMSE, R² scores
- **Data quality**: Record counts, validation results
- **Performance**: Resource utilization, cost tracking

---

## 🔄 Version Control

### Git Structure
```
.git/
├── 📁 objects/                # Git objects
├── 📁 refs/                   # References and branches
├── 📄 HEAD                    # Current branch reference
└── 📄 config                  # Git configuration
```

### Branch Strategy
- **main**: Production-ready code
- **develop**: Development branch
- **feature/***: Feature development
- **hotfix/***: Emergency fixes

---

## 📞 Support & Maintenance

### Documentation Locations
- **User Guide**: `docs/SETUP.md`
- **Technical Docs**: `docs/ARCHITECTURE.md`
- **Performance**: `docs/MODEL_PERFORMANCE.md`
- **Structure**: `docs/PROJECT_STRUCTURE.md`

### Maintenance Tasks
- **Weekly**: Performance monitoring review
- **Monthly**: Model retraining and validation
- **Quarterly**: Architecture review and optimization
- **Annually**: Security audit and compliance check

---

*Last updated: December 2024* 