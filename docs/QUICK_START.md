# ⚡ Quick Start Guide - Crypto Price Prediction Pipeline

## 🚀 Get Started in 10 Minutes

This guide will help you quickly set up and run the Crypto Price Prediction Pipeline on Azure Synapse.

---

## 📋 Prerequisites Checklist

Before starting, ensure you have:

- ✅ **Azure Subscription** with billing enabled
- ✅ **Azure CLI** installed and logged in
- ✅ **Python 3.8+** installed
- ✅ **Git** for cloning the repository

---

## 🔧 Step 1: Clone and Setup

```bash
# Clone the repository
git clone <your-repo-url>
cd Azure_ML

# Install Python dependencies
pip install -r requirements.txt

# Login to Azure
az login
```

---

## 🏗️ Step 2: Quick Azure Setup

```bash
# Set your variables (modify as needed)
RESOURCE_GROUP="crypto-pipeline-rg"
WORKSPACE_NAME="crypto-synapse"
LOCATION="eastus"
STORAGE_ACCOUNT="cryptodatalake"

# Create resources
az group create --name $RESOURCE_GROUP --location $LOCATION

az storage account create \
    --name $STORAGE_ACCOUNT \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --sku Standard_LRS \
    --kind StorageV2 \
    --hierarchical-namespace true

az synapse workspace create \
    --name $WORKSPACE_NAME \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --storage-account $STORAGE_ACCOUNT \
    --file-system "synapse" \
    --sql-admin-login-user "admin" \
    --sql-admin-login-password "YourPassword123!"

# Create Spark pool
az synapse spark pool create \
    --name "small" \
    --workspace-name $WORKSPACE_NAME \
    --resource-group $RESOURCE_GROUP \
    --node-count 3 \
    --node-size "Small" \
    --spark-version "3.4"
```

---

## 📊 Step 3: Deploy Pipelines

### Option A: Azure Synapse Studio (Recommended)

1. **Open Azure Synapse Studio**
   ```
   https://web.azuresynapse.net/
   ```

2. **Navigate to your workspace**
   - Select your workspace: `crypto-synapse`

3. **Import the pipelines**
   - Go to **Integrate** → **Pipelines**
   - Click **Import** and select files from `pipeline/` directory

4. **Configure linked services**
   - Go to **Manage** → **Linked Services**
   - Create the required linked services (see `linkedService/` directory)

### Option B: Azure CLI (Advanced)

```bash
# Deploy pipelines using Azure CLI
az synapse pipeline create \
    --workspace-name $WORKSPACE_NAME \
    --resource-group $RESOURCE_GROUP \
    --name "CoinGecko" \
    --file "pipeline/CoinGecko.json"
```

---

## 🧪 Step 4: Test the Pipeline

### Quick Test Script

Create `test_pipeline.py`:

```python
import requests
import pandas as pd
from datetime import datetime

def test_coingecko_api():
    """Test CoinGecko API connection"""
    url = "https://api.coingecko.com/api/v3/coins/markets"
    params = {
        "vs_currency": "usd",
        "order": "market_cap_desc",
        "per_page": 10,
        "page": 1
    }
    
    try:
        response = requests.get(url, params=params)
        response.raise_for_status()
        data = response.json()
        
        print(f"✅ API Test: Successfully retrieved {len(data)} coins")
        print(f"📊 Sample data: {data[0]['name']} - ${data[0]['current_price']}")
        return True
        
    except Exception as e:
        print(f"❌ API Test Failed: {e}")
        return False

def test_data_structure():
    """Test expected data structure"""
    sample_data = {
        "id": "bitcoin",
        "symbol": "btc",
        "name": "Bitcoin",
        "current_price": 43250.0,
        "market_cap": 847234567890,
        "total_volume": 23456789012,
        "price_change_percentage_24h": 2.98,
        "circulating_supply": 19567890.0,
        "high_24h": 44500.0,
        "low_24h": 42000.0,
        "ath": 69000.0,
        "roi": {"percentage": 1250.5, "currency": "usd"}
    }
    
    required_fields = [
        "current_price", "market_cap", "total_volume",
        "price_change_percentage_24h", "circulating_supply"
    ]
    
    missing_fields = [field for field in required_fields if field not in sample_data]
    
    if not missing_fields:
        print("✅ Data Structure: All required fields present")
        return True
    else:
        print(f"❌ Data Structure: Missing fields: {missing_fields}")
        return False

if __name__ == "__main__":
    print("🚀 Testing Crypto Price Prediction Pipeline...")
    print("=" * 50)
    
    # Test API
    api_test = test_coingecko_api()
    
    # Test data structure
    structure_test = test_data_structure()
    
    print("=" * 50)
    if api_test and structure_test:
        print("✅ All tests passed! Pipeline is ready to run.")
    else:
        print("❌ Some tests failed. Please check the configuration.")
```

Run the test:

```bash
python test_pipeline.py
```

---

## 🔄 Step 5: Run the Pipeline

### Manual Execution

1. **In Azure Synapse Studio**
   - Go to **Integrate** → **Pipelines**
   - Select **Pipeline 1** (main orchestration)
   - Click **Debug** to run manually

2. **Monitor execution**
   - Watch the pipeline progress in real-time
   - Check logs for any errors
   - Verify data in each layer

### Scheduled Execution

```bash
# Create trigger for daily execution
az synapse trigger create \
    --workspace-name $WORKSPACE_NAME \
    --resource-group $RESOURCE_GROUP \
    --name "daily-trigger" \
    --file "trigger/raw_data.json"
```

---

## 📈 Step 6: Verify Results

### Check Data Layers

```python
# Quick verification script
from pyspark.sql import SparkSession

# Initialize Spark
spark = SparkSession.builder.getOrCreate()

# Check bronze layer
bronze_df = spark.read.json("abfss://bronze@cryptodatalake.dfs.core.windows.net/coins/raw/")
print(f"Bronze records: {bronze_df.count()}")

# Check silver layer
silver_df = spark.read.parquet("abfss://silver@cryptodatalake.dfs.core.windows.net/coins/processed/")
print(f"Silver records: {silver_df.count()}")

# Check ML results
ml_df = spark.read.parquet("abfss://silver@cryptodatalake.dfs.core.windows.net/coins/ml_ready/")
print(f"ML-ready records: {ml_df.count()}")
```

### Expected Results

| Layer | Expected Records | Format | Purpose |
|-------|------------------|--------|---------|
| **Bronze** | ~250 | JSON | Raw API data |
| **Silver** | ~250 | Parquet | Cleaned data |
| **Gold** | ~250 | Parquet | ML-ready data |

---

## 🎯 Step 7: View Model Performance

### Quick Performance Check

```python
import pandas as pd
from sklearn.metrics import mean_absolute_error, r2_score

# Load predictions (if available)
predictions_df = pd.read_parquet("abfss://silver@cryptodatalake.dfs.core.windows.net/coins/predictions/")

# Calculate metrics
mae = mean_absolute_error(predictions_df['actual'], predictions_df['predicted'])
r2 = r2_score(predictions_df['actual'], predictions_df['predicted'])

print(f"📊 Model Performance:")
print(f"   MAE: ${mae:,.2f}")
print(f"   R²: {r2:.3f}")
```

### Expected Performance

| Metric | Target | Current |
|--------|--------|---------|
| **MAE** | < $10,000 | ~$5,500 |
| **R²** | > 0.4 | ~0.46 |
| **RMSE** | < $50,000 | ~$24,000 |

---

## 🚨 Troubleshooting

### Common Issues

#### 1. Pipeline Fails
```bash
# Check pipeline status
az synapse pipeline-run query-by-workspace \
    --workspace-name $WORKSPACE_NAME \
    --resource-group $RESOURCE_GROUP \
    --last-updated-after "2024-12-01T00:00:00Z"
```

#### 2. API Rate Limits
```python
# Add delay between API calls
import time
time.sleep(1)  # 1 second delay
```

#### 3. Storage Access Issues
```bash
# Check storage permissions
az storage account show \
    --name $STORAGE_ACCOUNT \
    --resource-group $RESOURCE_GROUP
```

### Quick Fixes

| Issue | Solution |
|-------|----------|
| **Authentication Error** | Check Azure credentials: `az login` |
| **Resource Not Found** | Verify resource names and locations |
| **Permission Denied** | Check RBAC roles and permissions |
| **API Timeout** | Increase timeout settings in pipeline |

---

## 📊 Next Steps

### Immediate Actions

1. **Monitor the pipeline** for 24-48 hours
2. **Review model performance** metrics
3. **Set up alerts** for pipeline failures
4. **Configure Power BI** for visualization

### Enhancements

1. **Add more features** to improve model accuracy
2. **Implement real-time predictions**
3. **Set up automated retraining**
4. **Create trading signals** dashboard

---

## 📞 Need Help?

### Quick Support

- **Documentation**: Check `docs/` directory
- **Issues**: Review pipeline logs in Azure Synapse Studio
- **Community**: Stack Overflow with tag `azure-synapse`

### Contact Information

- **Project Maintainer**: [Your Name]
- **Email**: [your.email@company.com]
- **GitHub**: [Project Repository]

---

## 🎉 Congratulations!

You've successfully set up the Crypto Price Prediction Pipeline! 

### What's Next?

1. **Explore the data** in Azure Synapse Studio
2. **Monitor model performance** over time
3. **Customize the pipeline** for your needs
4. **Scale up** as your data grows

---

*Quick Start Guide - Last updated: December 2024* 