# 🚀 Setup Guide - Crypto Price Prediction Pipeline

## Prerequisites

Before setting up the pipeline, ensure you have the following:

### Azure Requirements
- ✅ **Azure Subscription** with billing enabled
- ✅ **Azure Synapse Workspace** access
- ✅ **Azure Data Lake Gen2** storage account
- ✅ **Contributor permissions** on the resource group

### Development Environment
- ✅ **Python 3.8+** installed
- ✅ **Azure CLI** installed and configured
- ✅ **Git** for version control
- ✅ **VS Code** or similar IDE (recommended)

---

## 🏗️ Step 1: Azure Infrastructure Setup

### 1.1 Create Azure Synapse Workspace

```bash
# Login to Azure
az login

# Set variables
RESOURCE_GROUP="new_recurse_service"
WORKSPACE_NAME="alex2026"
LOCATION="eastus"
STORAGE_ACCOUNT="lakeiqbetim"

# Create resource group (if not exists)
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create storage account
az storage account create \
    --name $STORAGE_ACCOUNT \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --sku Standard_LRS \
    --kind StorageV2 \
    --hierarchical-namespace true

# Create Synapse workspace
az synapse workspace create \
    --name $WORKSPACE_NAME \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --storage-account $STORAGE_ACCOUNT \
    --file-system "synapse" \
    --sql-admin-login-user "sqladminuser" \
    --sql-admin-login-password "YourPassword123!"
```

### 1.2 Create Spark Pool

```bash
# Create Spark pool
az synapse spark pool create \
    --name "smalsize" \
    --workspace-name $WORKSPACE_NAME \
    --resource-group $RESOURCE_GROUP \
    --node-count 10 \
    --node-size "Small" \
    --spark-version "3.4"
```

### 1.3 Configure Data Lake Containers

```bash
# Get storage account key
STORAGE_KEY=$(az storage account keys list \
    --account-name $STORAGE_ACCOUNT \
    --resource-group $RESOURCE_GROUP \
    --query "[0].value" -o tsv)

# Create containers
az storage container create \
    --name "bronze" \
    --account-name $STORAGE_ACCOUNT \
    --account-key $STORAGE_KEY

az storage container create \
    --name "silver" \
    --account-name $STORAGE_ACCOUNT \
    --account-key $STORAGE_KEY

az storage container create \
    --name "gold" \
    --account-name $STORAGE_ACCOUNT \
    --account-key $STORAGE_KEY
```

---

## 🔧 Step 2: Linked Services Configuration

### 2.1 Azure Data Lake Storage

Navigate to **Azure Synapse Studio** → **Manage** → **Linked Services** and create:

```json
{
  "name": "AzureDataLakeStorage1",
  "type": "Microsoft.Synapse/workspaces/linkedservices",
  "properties": {
    "type": "AzureBlobFS",
    "typeProperties": {
      "url": "https://lakeiqbetim.dfs.core.windows.net"
    },
    "connectVia": {
      "referenceName": "AutoResolveIntegrationRuntime",
      "type": "IntegrationRuntimeReference"
    }
  }
}
```

### 2.2 HTTP Server (CoinGecko API)

```json
{
  "name": "HttpServer1",
  "type": "Microsoft.Synapse/workspaces/linkedservices",
  "properties": {
    "type": "HttpServer",
    "typeProperties": {
      "url": "https://api.coingecko.com",
      "enableServerCertificateValidation": true,
      "authenticationType": "Anonymous"
    },
    "connectVia": {
      "referenceName": "AutoResolveIntegrationRuntime",
      "type": "IntegrationRuntimeReference"
    }
  }
}
```

### 2.3 Azure ML Service

```json
{
  "name": "AzureMLService1",
  "type": "Microsoft.Synapse/workspaces/linkedservices",
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

---

## 📊 Step 3: Dataset Configuration

### 3.1 CoinGecko API Dataset

Create dataset `RestResource1.json`:

```json
{
  "name": "RestResource1",
  "properties": {
    "linkedServiceName": {
      "referenceName": "HttpServer1",
      "type": "LinkedServiceReference"
    },
    "parameters": {},
    "annotations": [],
    "type": "RestResource",
    "typeProperties": {
      "relativeUrl": "/api/v3/coins/markets",
      "requestMethod": "GET",
      "additionalHeaders": {
        "Accept": "application/json"
      },
      "requestBody": "",
      "paginationRules": {}
    },
    "schema": []
  }
}
```

### 3.2 Bronze Layer Dataset

Create dataset for bronze layer storage:

```json
{
  "name": "BronzeDataset",
  "properties": {
    "linkedServiceName": {
      "referenceName": "AzureDataLakeStorage1",
      "type": "LinkedServiceReference"
    },
    "parameters": {},
    "annotations": [],
    "type": "Json",
    "typeProperties": {
      "location": {
        "type": "AzureBlobFSLocation",
        "folderPath": "coins/raw",
        "fileSystem": "bronze"
      }
    },
    "schema": []
  }
}
```

---

## 🔄 Step 4: Pipeline Deployment

### 4.1 Deploy Bronze Layer Pipeline

1. Navigate to **Azure Synapse Studio** → **Integrate** → **Pipelines**
2. Create new pipeline named `CoinGecko`
3. Add **Copy Data** activity with:
   - **Source**: RestResource1 (CoinGecko API)
   - **Sink**: BronzeDataset (Data Lake)
   - **Mapping**: Automatic schema detection

### 4.2 Deploy Silver Layer Pipeline

1. Create pipeline named `coingecko_notebook`
2. Add **Synapse Notebook** activity
3. Configure notebook `data_transformation_silver.json`
4. Set Spark pool to `smalsize`

### 4.3 Deploy Gold Layer Pipeline

1. Create pipeline named `ML_coinGecko`
2. Add **Synapse Notebook** activity
3. Configure notebook `ML_CoinGecko.json`
4. Set Spark pool to `smalsize`

### 4.4 Deploy Main Orchestration Pipeline

1. Create pipeline named `Pipeline 1`
2. Add **Execute Pipeline** activities for each layer
3. Configure dependencies:
   - Bronze → Silver → Gold

---

## ⚙️ Step 5: Trigger Configuration

### 5.1 Create Scheduled Trigger

```json
{
  "name": "raw_data",
  "properties": {
    "annotations": [],
    "runtimeState": "Started",
    "pipelines": [
      {
        "pipelineReference": {
          "referenceName": "Pipeline 1",
          "type": "PipelineReference"
        }
      }
    ],
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

---

## 🧪 Step 6: Testing & Validation

### 6.1 Test Data Ingestion

```python
# Test bronze layer data
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()

# Read bronze data
bronze_df = spark.read.json("abfss://bronze@lakeiqbetim.dfs.core.windows.net/coins/raw/")
print(f"Bronze records: {bronze_df.count()}")
bronze_df.show(5)
```

### 6.2 Test Data Transformation

```python
# Test silver layer processing
silver_df = spark.read.parquet("abfss://silver@lakeiqbetim.dfs.core.windows.net/coins/processed/")
print(f"Silver records: {silver_df.count()}")
silver_df.show(5)
```

### 6.3 Test ML Pipeline

```python
# Test ML model
import pandas as pd
from sklearn.metrics import mean_absolute_error, r2_score

# Load ML-ready data
ml_df = pd.read_parquet("abfss://silver@lakeiqbetim.dfs.core.windows.net/coins/ml_ready/")
print(f"ML-ready records: {len(ml_df)}")
print(f"Features: {ml_df.columns.tolist()}")
```

---

## 🔍 Step 7: Monitoring Setup

### 7.1 Azure Monitor Configuration

```bash
# Enable diagnostic settings
az monitor diagnostic-settings create \
    --name "synapse-diagnostics" \
    --resource-group $RESOURCE_GROUP \
    --resource-name $WORKSPACE_NAME \
    --resource-type "Microsoft.Synapse/workspaces" \
    --logs '[{"category": "SynapseRbacOperations", "enabled": true}]' \
    --metrics '[{"category": "AllMetrics", "enabled": true}]' \
    --storage-account $STORAGE_ACCOUNT
```

### 7.2 Log Analytics Workspace

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
    --resource-group $RESOURCE_GROUP \
    --workspace-name "crypto-pipeline-logs"
```

---

## 🚨 Step 8: Alerting Configuration

### 8.1 Pipeline Failure Alerts

```bash
# Create action group
az monitor action-group create \
    --name "pipeline-alerts" \
    --resource-group $RESOURCE_GROUP \
    --short-name "pipeline-alerts" \
    --action email admin@yourcompany.com "Pipeline Admin"

# Create alert rule
az monitor metrics alert create \
    --name "pipeline-failure-alert" \
    --resource-group $RESOURCE_GROUP \
    --scopes "/subscriptions/c13fe431-45c7-45ed-aafb-780b48928c18/resourceGroups/new_recurse_service/providers/Microsoft.Synapse/workspaces/alex2026" \
    --condition "total PipelineRunsFailed > 0" \
    --window-size "PT5M" \
    --evaluation-frequency "PT5M" \
    --action "/subscriptions/c13fe431-45c7-45ed-aafb-780b48928c18/resourceGroups/new_recurse_service/providers/Microsoft.Insights/actionGroups/pipeline-alerts"
```

---

## 📋 Step 9: Validation Checklist

### Infrastructure
- [ ] Azure Synapse Workspace created
- [ ] Spark pool configured
- [ ] Data Lake containers created
- [ ] Linked services configured

### Pipelines
- [ ] Bronze layer pipeline deployed
- [ ] Silver layer pipeline deployed
- [ ] Gold layer pipeline deployed
- [ ] Main orchestration pipeline deployed
- [ ] Trigger configured

### Data Flow
- [ ] CoinGecko API connection tested
- [ ] Bronze layer data ingestion working
- [ ] Silver layer transformation working
- [ ] Gold layer ML processing working

### Monitoring
- [ ] Diagnostic settings enabled
- [ ] Log Analytics workspace created
- [ ] Alert rules configured
- [ ] Performance monitoring active

---

## 🔧 Troubleshooting

### Common Issues

#### 1. Pipeline Failures
```bash
# Check pipeline runs
az synapse pipeline-run query-by-workspace \
    --workspace-name $WORKSPACE_NAME \
    --resource-group $RESOURCE_GROUP \
    --last-updated-after "2024-12-01T00:00:00Z" \
    --last-updated-before "2024-12-02T00:00:00Z"
```

#### 2. Spark Pool Issues
```bash
# Check Spark pool status
az synapse spark pool show \
    --name "smalsize" \
    --workspace-name $WORKSPACE_NAME \
    --resource-group $RESOURCE_GROUP
```

#### 3. Storage Access Issues
```bash
# Test storage access
az storage blob list \
    --container-name "bronze" \
    --account-name $STORAGE_ACCOUNT \
    --account-key $STORAGE_KEY
```

### Performance Optimization

#### 1. Spark Pool Tuning
```json
{
  "driverMemory": "28g",
  "driverCores": 4,
  "executorMemory": "28g",
  "executorCores": 4,
  "numExecutors": 2
}
```

#### 2. Data Partitioning
```python
# Optimize partitioning
df.write.partitionBy("date").parquet("abfss://silver@lakeiqbetim.dfs.core.windows.net/coins/processed/")
```

---

## 📞 Support

### Getting Help
- **Azure Documentation**: [Azure Synapse Analytics](https://docs.microsoft.com/en-us/azure/synapse-analytics/)
- **CoinGecko API**: [API Documentation](https://www.coingecko.com/en/api/documentation)
- **Community Support**: [Stack Overflow](https://stackoverflow.com/questions/tagged/azure-synapse)

### Contact Information
- **Project Maintainer**: [Your Name]
- **Email**: [your.email@company.com]
- **GitHub Issues**: [Project Repository Issues]

---

*Last updated: December 2024* 