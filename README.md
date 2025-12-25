# 🌐 GCP Serverless API Data Pipeline

A production-ready, end-to-end serverless data ingestion pipeline built on GCP.

---

## 📋 Overview

This project shows a fully serverless data engineering pipeline that ingests data from an Frankfurter (Exchange Rates) API, stores it in to blob storage, and loads it into a warehouse / table.

<img width="1844" height="813" alt="image" src="https://github.com/user-attachments/assets/897fbcfc-1ba7-453f-bb5f-89ff975ca51a" />

**The pipeline:**
- Cloud Scheduler: schedules a Functions code to run every hour
- Functions (Cloud Run): Python code to call and return Frankfurter API in JSON
- Google Cloud Storage (GCS): Stores JSON 
- BigQuery: scheduled to LOAD data from GCS every hour into a table

This architecture mirrors common AWS patterns (Lambda + EventBridge + S3 + Snowflake) but uses **GCP-native services** for a cloud-agnostic skill demonstration.

---

## 🏗️ Architecture

**Data Flow:**

```
Cloud Scheduler (cron) → Cloud Function (Python) → Cloud Storage (JSON) → BigQuery (Analytics)
```

---

## 🛠️ Technologies Used

| Technology | Purpose |
|------------|---------|
| **Python 3** | Core programming language |
| **Cloud Shell** | API call testing |
| **Google Cloud Functions** | Serverless compute |
| **Cloud Scheduler** | Automated job orchestration |
| **Cloud Storage** | Raw data lake (bronze layer) |
| **BigQuery** | Data warehouse and analytics |

---

## 🚀 Project Walkthrough

### **Phase 1: API Selection & Local Testing**

**Objective:** Select a reliable public API and validate response structure.

**Actions:**
- Chose an API returning JSON data: [Frankfurter API](https://frankfurter.dev/)
- Tested response format and data consistency using [Cloud Shell](https://github.com/roshkhan2000/GCP-Serverless-API-Data-Pipeline/blob/main/Cloud%20Shell%20API%20Test%20Call)

---

### **Phase 2: Cloud Function (Serverless Ingestion)**

**Objective:** Build a serverless Python function to call the API and store results.

**Key Features:**
- HTTP-triggered Cloud Function [script](https://github.com/roshkhan2000/GCP-Serverless-API-Data-Pipeline/blob/main/Cloud%20Functions%20Python%20Code) with described [requirements](https://github.com/roshkhan2000/GCP-Serverless-API-Data-Pipeline/blob/main/Cloud%20Functions%20requirements.txt) deployed with Python 3
- Uses `requests` library for API calls
- Adds timezone-aware UTC timestamp at ingestion
- Writes JSON directly to Cloud Storage (blob)

**Output:** Each run creates a file at:
```
gs://<bucket-name>/raw/exchange_rates/<timestamp>.json
```

---

### **Phase 3: Cloud Storage (Raw Data Layer & Blob Storage)**

**Objective:** Preserve unmodified API responses for reprocessing and audit trails.

**Design Decisions:**
- Raw data stored under `/raw/` prefix
- One JSON file per API call
- Exact API response format preserved

This represents the **bronze layer** in a medallion architecture pattern.

---

### **Phase 4: Scheduling with Cloud Scheduler**

**Objective:** Automate the ingestion process to run hourly.

**Configuration:**
- **Cron Expression:** `0 * * * *` (every hour)
- **Target:** Cloud Function HTTP endpoint
- **Authentication:** OIDC via service account

This replaces AWS EventBridge-style scheduling with GCP-native tooling.

---

### **Phase 5: BigQuery Raw Table**

**Objective:** Create a schema-controlled table for raw data ingestion.

**Table Schema:**
```sql
CREATE TABLE exchange_rates_raw (
  ingestion_timestamp TIMESTAMP,
  base_currency STRING,
  date DATE,
  rates JSON
);
```

**Key Considerations:**
- Explicit schema defined (no auto-detection)
- `rates` stored as JSON for downstream flexibility
- `ingestion_timestamp` generated at load time for tracking

---

### **Phase 6: Scheduled Load into BigQuery**

**Objective:** Automatically load JSON files from Cloud Storage into BigQuery.

**Load Query:**
```sql
LOAD DATA INTO `project.dataset.exchange_rates_raw`
FROM FILES (
  format = 'JSON',
  uris = ['gs://<bucket-name>/raw/exchange_rates/*.json']
);
```

**Important Notes:**
- Load jobs **append by default**
- BigQuery does not track previously loaded files
- This is intentional for raw ingestion design

---

### **Handling Duplicates**

Because scheduled loads append data, duplicate rows can occur.

**Strategy:**
- Allow duplicates in the raw layer
- Deduplicate downstream using SQL window functions

**Example Deduplication Query:**
```sql
SELECT *
FROM (
  SELECT *,
         ROW_NUMBER() OVER (
           PARTITION BY ingestion_timestamp, base_currency, date
           ORDER BY ingestion_timestamp DESC
         ) AS rn
  FROM exchange_rates_raw
)
WHERE rn = 1;
```
---

## 🔮 Possible Extensions

- [ ] Partition BigQuery tables by `ingestion_date` for query performance
- [ ] Flatten `rates` JSON into a dimensional fact table
- [ ] Add data quality checks and alerting
- [ ] Move processed files to a `/processed/` prefix
- [ ] Build Looker/Data Studio dashboards on curated data
- [ ] Implement CI/CD with Cloud Build
- [ ] Add unit tests and integration tests
