# 🚀 Crypto Price Prediction Pipeline – Azure Synapse + ML

## 📋 Table of Contents
- [Project Overview](#-project-overview)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [Setup & Installation](#-setup--installation)
- [Pipeline Flow](#-pipeline-flow)
- [Model Performance](#-model-performance)
- [Features](#-features)
- [Business Value](#-business-value)
- [Next Steps](#-next-steps)
- [Contributing](#-contributing)

---

## 🔍 Project Overview

This project implements a complete **machine learning pipeline for predicting cryptocurrency prices** using public data from the CoinGecko API. It leverages a **Medallion architecture** (bronze, silver, gold) within Azure Synapse and integrates **PySpark** and **scikit-learn/XGBoost** for modeling.

### 🎯 Key Objectives
- **Automated data ingestion** from CoinGecko API
- **Real-time data processing** with PySpark
- **ML model training** with XGBoost and hyperparameter tuning
- **Production-ready pipeline** orchestration in Azure Synapse
- **Scalable architecture** for handling large crypto datasets

---

## 🏗️ Architecture

### Medallion Architecture
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   BRONZE LAYER  │    │   SILVER LAYER  │    │    GOLD LAYER   │
│                 │    │                 │    │                 │
│ • Raw JSON data │───▶│ • Cleaned data  │───▶│ • ML-ready data │
│ • CoinGecko API │    │ • Feature eng.  │    │ • Aggregated    │
│ • No schema     │    │ • Parquet format│    │ • Business logic│
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Pipeline Components
1. **Data Ingestion (Bronze)**: CoinGecko API → Azure Data Lake Gen2
2. **Data Transformation (Silver)**: PySpark processing → Feature engineering
3. **Machine Learning (Gold)**: XGBoost model → Predictions & insights

---

## 🛠️ Tech Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Data Lake** | Azure Data Lake Gen2 | Raw and processed data storage |
| **Orchestration** | Azure Synapse Pipelines | Workflow management |
| **Processing** | PySpark | Data transformation |
| **ML Framework** | scikit-learn + XGBoost | Model training |
| **Hyperparameter Tuning** | GridSearchCV | Model optimization |
| **Storage Format** | Parquet | Optimized data storage |
| **API Integration** | CoinGecko REST API | Data source |

---

## 📁 Project Structure

```
Azure_ML/
├── 📁 pipeline/                    # Azure Synapse pipeline definitions
│   ├── Pipeline 1.json            # Main orchestration pipeline
│   ├── CoinGecko.json             # Bronze layer pipeline
│   ├── coingecko_notebook.json    # Silver layer pipeline
│   └── ML_coinGecko.json          # Gold layer pipeline
├── 📁 notebook/                    # Jupyter notebooks for ML
│   ├── ML_CoinGecko.json          # Main ML training notebook
│   ├── data_transformation_silver.json
│   ├── silver_transform.json
│   └── gold_layer.json
├── 📁 dataset/                     # Dataset definitions
│   ├── RestResource1.json         # CoinGecko API dataset
│   └── [other dataset configs]
├── 📁 linkedService/               # Azure service connections
│   ├── AzureDataLakeStorage1.json
│   ├── AzureMLService1.json
│   └── HttpServer1.json
├── 📁 trigger/                     # Pipeline triggers
│   └── raw_data.json
├── 📁 sqlscript/                   # SQL scripts (if any)
├── 📁 integrationRuntime/          # Integration runtime configs
├── 📁 credential/                  # Credential management
└── 📄 README.md                    # This documentation
```

---

## 🚀 Setup & Installation

### Prerequisites
- Azure Subscription
- Azure Synapse Workspace
- Azure Data Lake Gen2 Storage Account
- Python 3.8+ with required packages

### Required Python Packages
```bash
pip install pandas numpy scikit-learn xgboost pyspark matplotlib seaborn
```

### Azure Services Setup
1. **Create Azure Synapse Workspace**
2. **Configure Data Lake Gen2 Storage**
3. **Set up Linked Services** (see `linkedService/` directory)
4. **Deploy Pipelines** (see `pipeline/` directory)

---

## 🔄 Pipeline Flow

### 1. Bronze Layer (Data Ingestion)
```json
Pipeline: CoinGecko.json
- Source: CoinGecko API (coins/markets?vs_currency=usd)
- Destination: Azure Data Lake Gen2 (bronze container)
- Format: Raw JSON
- Frequency: Triggered (see trigger/raw_data.json)
```

### 2. Silver Layer (Data Transformation)
```json
Pipeline: coingecko_notebook.json
- Processing: PySpark notebook
- Features: Cleaning, feature engineering
- Output: Parquet format in silver container
- Features added:
  - price_volatility (high_24h - low_24h)
  - ath_pct (percent from ATH)
  - roi_percentage (from CoinGecko)
```

### 3. Gold Layer (Machine Learning)
```json
Pipeline: ML_coinGecko.json
- Model: XGBoost with GridSearchCV
- Target: current_price prediction
- Metrics: MAE, RMSE, R²
- Output: Trained model + predictions
```

---

## 📊 Model Performance

### Current Results
| Metric | Score | Interpretation |
|--------|-------|----------------|
| **MAE** | 5,554 USD | Average absolute error |
| **RMSE** | 23,923 USD | Root mean square error |
| **R²** | 0.46 | 46% variance explained |

### Model Comparison
- **Random Forest**: Baseline model
- **XGBoost**: Optimized with GridSearchCV
- **Stacking**: Ensemble of multiple models

### Feature Importance
Top features by importance:
1. `market_cap` - Market capitalization
2. `total_volume` - 24h trading volume
3. `circulating_supply` - Available coins
4. `price_change_percentage_24h` - Price momentum
5. `roi_percentage` - Return on investment

---

## 🎯 Features

### Input Features
| Feature | Description | Source |
|---------|-------------|--------|
| `market_cap` | Total market capitalization | CoinGecko API |
| `total_volume` | 24h traded volume | CoinGecko API |
| `price_change_percentage_24h` | 24h price variation | CoinGecko API |
| `circulating_supply` | Total coins in circulation | CoinGecko API |
| `roi_percentage` | Return on Investment | CoinGecko API |
| `price_volatility` | `high_24h - low_24h` | Calculated |
| `ath_pct` | Percent drop from all-time high | Calculated |

### Target Variable
- `current_price`: Current cryptocurrency price in USD

---

## 💼 Business Value

### ROI Benefits
- 🚀 **Automated data ingestion** – No manual data pulling
- 🧠 **Actionable insights** – Predicts crypto prices with explainability
- 💡 **Production-ready** – Modular pipeline with orchestrated workflows
- ⏱️ **Time-to-insight** – Full retraining takes minutes
- 📊 **Dashboard ready** – Great base for monitoring systems

### Use Cases
- **Trading signals** generation
- **Portfolio optimization** insights
- **Risk assessment** for crypto investments
- **Market trend** analysis
- **Alert systems** for price movements

---

## 🔮 Next Steps

### Immediate Improvements
- [ ] **Add CoinGecko pagination** to increase training data volume
- [ ] **Implement new features**: ATH, ATL, sentiment scores
- [ ] **Deploy model** using Azure ML endpoints
- [ ] **Connect Power BI** for monitoring predictions vs. actuals

### Advanced Features
- [ ] **Time series forecasting** with Prophet or LSTM
- [ ] **Real-time predictions** with streaming data
- [ ] **Multi-currency support** (EUR, GBP, etc.)
- [ ] **Sentiment analysis** integration
- [ ] **Automated model retraining** schedules

### Production Enhancements
- [ ] **Model versioning** and A/B testing
- [ ] **Performance monitoring** and alerting
- [ ] **Data quality checks** and validation
- [ ] **Cost optimization** for compute resources

---

## 🤝 Contributing

### Development Guidelines
1. **Follow the Medallion architecture** pattern
2. **Use consistent naming** conventions
3. **Document all changes** in notebooks
4. **Test pipelines** before deployment
5. **Monitor model performance** regularly

### Code Standards
- Python: PEP 8 compliance
- JSON: Proper formatting and validation
- Documentation: Clear and comprehensive
- Testing: Unit tests for critical functions

---

## 📞 Support

For questions or issues:
- Check the pipeline logs in Azure Synapse
- Review the notebook outputs for errors
- Validate the data quality in each layer
- Monitor the model performance metrics

---

## 📄 License

This project is for educational and research purposes. Please ensure compliance with CoinGecko API terms of service.

---

*Last updated: December 2024*