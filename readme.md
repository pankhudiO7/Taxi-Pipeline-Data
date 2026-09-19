# Taxi Pipeline Data — NYC Yellow Taxi Medallion Lakehouse

A local, containerized data lakehouse pipeline that ingests NYC Yellow Taxi trip
data through a **bronze → silver → gold** (medallion) architecture using
**Apache Airflow**, **Apache Spark**, and **Apache Iceberg**.

> Built and run locally for CDA500 as a hands-on demonstration of asset-driven
> orchestration, idempotent ingestion, and the Write-Audit-Publish (WAP) data
> quality pattern.

---

## Architecture

```
Incoming parquet file
        │
        ▼
┌───────────────────┐
│  Mock Lambda /     │  watches data/incoming/ for new files + manifests,
│  SFTP watcher      │  emits an Airflow asset event
└─────────┬──────────┘
          ▼
┌───────────────────┐
│ 1_bronze_ingestion │  raw ingest • SHA-256 checksum • idempotency ledger
└─────────┬──────────┘
          ▼ (asset event: warehouse/bronze/yellow_taxi_raw)
┌───────────────────────┐
│ 2_silver_wap_purification │  Write-Audit-Publish: candidate → validate → publish
└─────────┬──────────────┘      bad rows routed to a quarantine table
          ▼ (asset event: warehouse/silver/yellow_taxi_trips)
     ┌────┴─────┐
     ▼          ▼
┌─────────┐ ┌───────────────────┐
│ 3a_gold │ │ 3b_gold_ml_        │
│ _bi_marts│ │ observations       │
└─────────┘ └───────────────────┘
  BI-ready       ML-ready feature
  aggregates     rows
```

All DAGs are **asset-triggered**, not manually scheduled — dropping a new
month of data and notifying Airflow is enough to run the entire pipeline
end-to-end automatically.

## Tech stack

| Layer | Tool |
|---|---|
| Orchestration | Apache Airflow (asset/dataset-driven scheduling) |
| Processing | Apache Spark (PySpark) |
| Table format | Apache Iceberg (Hadoop catalog) |
| Metadata DB | Postgres |
| Exploration | Jupyter Notebook |
| Infra | Docker Compose |

## Project layout

```
dags/               Airflow DAGs (bronze, silver, gold x2, hello_world sanity check)
jobs/                Spark jobs run by each DAG (bronze_ingest, silver_wap, gold_bi, gold_ml)
airflow-image/       Custom Airflow+Spark+Iceberg Dockerfile
config/local.yml      Catalog + storage configuration
notebooks/           Jupyter notebooks for EDA and Iceberg queries
sftp_mock_daemon.py   Simulated file-arrival watcher (mimics an S3/Lambda trigger)
drop_month.sh         Helper to stage a month of taxi data + generate its manifest
init.sh              One-time local environment bootstrap (.env, auth, folders)
docker-compose.yaml   Full local stack definition
```

## Running it locally

```bash
git clone https://github.com/pankhudiO7/Taxi-Pipeline-Data.git
cd Taxi-Pipeline-Data

# 1. Bootstrap local env, folders, and auth
chmod +x init.sh
./init.sh

# 2. Build the custom Airflow/Spark/Iceberg image
docker compose build

# 3. Start the stack
docker compose up
```

Once containers are healthy:

- **Airflow UI** → http://localhost:8080 (`admin` / `admin`)
- **Jupyter** → http://localhost:8888 (token `dev_token`)

### Feeding it data

```bash
# Download a month of NYC Yellow Taxi trip data, e.g.:
curl -L -o ~/Downloads/yellow_tripdata_2026-04.parquet \
  https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2026-04.parquet

cp ~/Downloads/yellow_tripdata_2026-04.parquet ./data/incoming/
./drop_month.sh 2026 4
```

Then start the mock watcher so it notifies Airflow of the new manifest:

```bash
docker exec -it mock_lambda_watcher sh -c \
  "pip install --no-cache-dir watchdog requests && python /app/sftp_mock_daemon.py"
```

Airflow picks up the asset event, runs `1_bronze_ingestion`, and the rest of
the pipeline cascades automatically.

> **Note:** the `.env` file is intentionally not committed to this repo.
> Running `init.sh` regenerates it locally with dev-only credentials — do not
> reuse these values outside of local/teaching use.

## Screenshots

**Airflow home — health checks, pool slots, run history**

blob:https://claude.ai/cda26815-6cf0-4940-9fd4-cc576585e33a<img width="2940" height="1912" alt="image" src="https://github.com/user-attachments/assets/aa18b9a7-b728-4fd1-9140-0035e9cbf23a" />



**All five DAGs completed successfully end-to-end**

blob:https://claude.ai/951ea996-ec87-4a15-b9e4-520d60392a31<img width="2940" height="1912" alt="image" src="https://github.com/user-attachments/assets/49054586-af33-4fc8-aeb0-5163373face6" />


The run above shows the full cascade: `1_bronze_ingestion` →
`2_silver_wap_purification` → `3a_gold_bi_marts` and `3b_gold_ml_observations`
firing in sequence off of asset events, with no manual triggering after the
initial data drop.

## What this project demonstrates

- **Asset/event-driven orchestration** instead of fixed cron schedules
- **Idempotent ingestion** via file checksums and an ingestion ledger, safe
  to re-run without duplicating data
- **Write-Audit-Publish (WAP)** — data is validated in a candidate table
  before being published to the trusted silver table; invalid rows are
  quarantined rather than dropped or silently merged
- **Lakehouse table format (Iceberg)** on top of local storage, partitioned
  for efficient querying
- **Simulated cloud event trigger** — the mock SFTP/Lambda watcher mimics how
  a production pipeline would react to a file landing in S3
