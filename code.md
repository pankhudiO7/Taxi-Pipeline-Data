## Init

`./init.sh`


```bash
#!/usr/bin/env bash
set -euo pipefail

# -----------------------------------------------------------------------------
# Local teaching environment initializer
# Purpose:
# - Create a simple project layout
# - Create a simple Airflow auth password file
# - Create a local config file for the data engineering workflow
# - Create a .env file used by docker-compose.yml
#
# This is intentionally simple for teaching.
# Do not treat these credentials as production secrets.
# -----------------------------------------------------------------------------

AIRFLOW_ADMIN_USERNAME="admin"
AIRFLOW_ADMIN_PASSWORD="admin"

POSTGRES_USER="airflow_user"
POSTGRES_PASSWORD="airflow_pass"
POSTGRES_DB="airflow_metadata"

AIRFLOW_IMAGE_NAME="airflow-spark-iceberg-teaching:3.2.2"

# On Linux, using the host UID avoids root-owned files in mounted folders.
# On macOS/Windows Docker Desktop this is usually harmless.
AIRFLOW_UID="$(id -u)"

# Stable development keys for repeatable classroom behavior.
# Replace these for any non-local/non-teaching use.
AIRFLOW_FERNET_KEY="81HqDtbqAywKSOumSha3BhWNOdQ26slT6K0YaZeZyPs="
AIRFLOW_API_SECRET_KEY="dev_api_secret_key_change_me_for_nonlocal_use_123456"
AIRFLOW_JWT_SECRET="dev_jwt_secret_change_me_for_nonlocal_use_1234567890"
JUPYTER_TOKEN="dev_token"

echo "Creating local teaching project folders..."

mkdir -p \
  ./airflow-image \
  ./dags \
  ./jobs \
  ./sql \
  ./config \
  ./data/incoming/taxi_data_drop \
  ./data/warehouse \
  ./logs \
  ./airflow_auth \
  ./notebooks

echo "Creating Simple Auth Manager password file..."

cat > ./airflow_auth/simple_auth_manager_passwords.json.generated <<EOF
{"${AIRFLOW_ADMIN_USERNAME}": "${AIRFLOW_ADMIN_PASSWORD}"}
EOF

chmod 600 ./airflow_auth/simple_auth_manager_passwords.json.generated

echo "Creating local data engineering configuration..."

cat > ./config/local.yml <<'EOF'
environment: local

catalog:
  spark_catalog_name: nyc
  logical_name: nyc_catalog
  type: hadoop
  warehouse_path: /opt/airflow/data/warehouse

storage:
  incoming_directory: /opt/airflow/data/incoming
  warehouse_directory: /opt/airflow/data/warehouse

assets:
  incoming_taxi_data_uri: file:///opt/airflow/data/incoming/taxi_data_drop
  bronze_taxi_data_uri: iceberg://bronze.yellow_taxi_raw
  silver_taxi_data_uri: iceberg://silver.yellow_taxi_trips
EOF

echo "Creating docker-compose .env file..."

cat > ./.env <<EOF
AIRFLOW_UID=${AIRFLOW_UID}
AIRFLOW_IMAGE_NAME=${AIRFLOW_IMAGE_NAME}

POSTGRES_USER=${POSTGRES_USER}
POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
POSTGRES_DB=${POSTGRES_DB}

AIRFLOW_ADMIN_USERNAME=${AIRFLOW_ADMIN_USERNAME}
AIRFLOW_ADMIN_PASSWORD=${AIRFLOW_ADMIN_PASSWORD}

AIRFLOW_FERNET_KEY=${AIRFLOW_FERNET_KEY}
AIRFLOW_API_SECRET_KEY=${AIRFLOW_API_SECRET_KEY}
AIRFLOW_JWT_SECRET=${AIRFLOW_JWT_SECRET}

JUPYTER_TOKEN=${JUPYTER_TOKEN}
EOF

chmod 600 ./.env

# Keep most teaching folders easy to edit from the host.
chmod -R u+rwX,g+rwX \
  ./dags \
  ./jobs \
  ./sql \
  ./config \
  ./data \
  ./logs \
  ./notebooks \
  ./airflow-image

echo
echo "Local curriculum lakehouse structure created successfully."
echo
echo "Airflow login:"
echo "  username: ${AIRFLOW_ADMIN_USERNAME}"
echo "  password: ${AIRFLOW_ADMIN_PASSWORD}"
echo
echo "Jupyter token:"
echo "  ${JUPYTER_TOKEN}"
echo
echo "Next steps:"
echo "  1. Put the Dockerfile in ./airflow-image/Dockerfile"
echo "  2. Put docker-compose.yml in the project root"
echo "  3. Run: docker compose build"
echo "  4. Run: docker compose up"
```



## Docker file

`./airflow-image/Dockerfile`

```dockerfile
FROM apache/airflow:3.2.2-python3.11

ARG ICEBERG_VERSION=1.10.0

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    openjdk-17-jre-headless \
    curl \
    ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /opt/java \
    && JAVA_HOME_ACTUAL="$(dirname "$(dirname "$(readlink -f "$(command -v java)")")")" \
    && ln -s "${JAVA_HOME_ACTUAL}" /opt/java/openjdk \
    && /opt/java/openjdk/bin/java -version

ENV JAVA_HOME=/opt/java/openjdk
ENV PATH="${JAVA_HOME}/bin:${PATH}"

USER airflow

# Do not reinstall apache-airflow.
# The base image already contains Airflow 3.2.2.
#
# psycopg[binary] matches the SQLAlchemy URL:
# postgresql+psycopg://...
#
# PySpark 3.5.6 + Iceberg 1.10.0 mirrors AWS Glue 5.1 closely.
#
# Jupyter/IPython are included for the optional notebook profile in docker-compose.
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir \
    "psycopg[binary]==3.3.4" \
    "pandas==2.2.3" \
    "pyarrow==16.1.0" \
    "duckdb==1.1.3" \
    "pyspark==3.5.6" \
    "pyyaml==6.0.1" \
    "ipython==8.31.0" \
    "jupyter==1.1.1"

USER root

ENV SPARK_HOME=/home/airflow/.local/lib/python3.11/site-packages/pyspark
ENV ICEBERG_VERSION=${ICEBERG_VERSION}
ENV ICEBERG_SPARK_JAR=${SPARK_HOME}/jars/iceberg-spark-runtime-3.5_2.12-${ICEBERG_VERSION}.jar

# Download the Iceberg runtime JAR that matches:
# - Spark 3.5.x
# - Scala 2.12
# - Iceberg 1.10.0
RUN test -d "${SPARK_HOME}/jars" \
    && curl -fL \
    "https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-spark-runtime-3.5_2.12/${ICEBERG_VERSION}/iceberg-spark-runtime-3.5_2.12-${ICEBERG_VERSION}.jar" \
    -o "${ICEBERG_SPARK_JAR}" \
    && test -s "${ICEBERG_SPARK_JAR}" \
    && chmod 0644 "${ICEBERG_SPARK_JAR}" \
    && ls -lh "${ICEBERG_SPARK_JAR}"

USER airflow

RUN python -c "import pyspark, yaml, pandas, pyarrow, duckdb, psycopg, IPython, notebook; print('Airflow + PySpark + Iceberg + Jupyter teaching image dependencies OK')"
```


## docker-compose file


`./docker-compose.yaml`

```yaml
x-airflow-common-env: &airflow-common-env
  AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg://airflow_user:airflow_pass@airflow-metadata-db:5432/airflow_metadata
  AIRFLOW__CORE__LOAD_EXAMPLES: "False"
  AIRFLOW__CORE__EXECUTOR: LocalExecutor

  # Teaching-only local secrets.
  AIRFLOW__CORE__FERNET_KEY: 81HqDtbqAywKSOumSha3BhWNOdQ26slT6K0YaZeZyPs=
  AIRFLOW__API__SECRET_KEY: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
  AIRFLOW__API_AUTH__JWT_SECRET: super_secret_shared_jwt_token_must_be_long_64_bytes_abc123xyz_padding

  # Simple Auth Manager for local teaching use.
  AIRFLOW__CORE__SIMPLE_AUTH_MANAGER_USERS: admin:admin
  AIRFLOW__CORE__SIMPLE_AUTH_MANAGER_PASSWORDS_FILE: /opt/airflow/auth/simple_auth_manager_passwords.json.generated

  JAVA_HOME: /opt/java/openjdk
  SPARK_HOME: /home/airflow/.local/lib/python3.11/site-packages/pyspark
  PYTHONPATH: /opt/airflow

x-airflow-common-volumes: &airflow-common-volumes
  - ./dags:/opt/airflow/dags:rw
  - ./jobs:/opt/airflow/jobs:rw
  - ./sql:/opt/airflow/sql:rw
  - ./config:/opt/airflow/config:rw
  - ./data:/opt/airflow/data:rw
  - ./logs:/opt/airflow/logs:rw
  - ./airflow_auth:/opt/airflow/auth:rw
  - ./notebooks:/opt/airflow/notebooks:rw

services:
  airflow-metadata-db:
    image: postgres:17-alpine
    container_name: airflow_metadata_postgres
    environment:
      - POSTGRES_DB=airflow_metadata
      - POSTGRES_USER=airflow_user
      - POSTGRES_PASSWORD=airflow_pass
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - platform_network
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U airflow_user -d airflow_metadata" ]
      interval: 5s
      timeout: 5s
      retries: 5

  airflow-init:
    build:
      context: ./airflow-image
      dockerfile: Dockerfile
    image: airflow-spark-iceberg-teaching:3.2.2-python3.11
    container_name: airflow_init_job
    depends_on:
      airflow-metadata-db:
        condition: service_healthy
    environment:
      <<: *airflow-common-env
    volumes: *airflow-common-volumes
    command: >
      bash -c '
        mkdir -p /opt/airflow/auth /opt/airflow/data/incoming /opt/airflow/data/warehouse /opt/airflow/logs &&
        if [ ! -f /opt/airflow/auth/simple_auth_manager_passwords.json.generated ]; then
          echo "{\"admin\": \"admin\"}" > /opt/airflow/auth/simple_auth_manager_passwords.json.generated;
        fi &&
        airflow db migrate
      '
    networks:
      - platform_network
    restart: "no"

  airflow-webserver:
    image: airflow-spark-iceberg-teaching:3.2.2-python3.11
    pull_policy: never
    container_name: airflow_webserver
    depends_on:
      airflow-init:
        condition: service_completed_successfully
    environment:
      <<: *airflow-common-env
    ports:
      - "8080:8080"
    volumes: *airflow-common-volumes
    command: airflow api-server
    networks:
      - platform_network
    healthcheck:
      test: [ "CMD-SHELL", "python -c 'import urllib.request,sys; sys.exit(0 if urllib.request.urlopen(\"http://localhost:8080/api/v2/monitor/health\", timeout=3).getcode() == 200 else 1)'" ]
      interval: 5s
      timeout: 5s
      retries: 20
    restart: always

  airflow-scheduler:
    image: airflow-spark-iceberg-teaching:3.2.2-python3.11
    pull_policy: never
    container_name: airflow_scheduler
    depends_on:
      airflow-init:
        condition: service_completed_successfully
      airflow-webserver:
        condition: service_healthy
    environment:
      <<: *airflow-common-env
      AIRFLOW__CORE__EXECUTION_API_SERVER_URL: http://airflow-webserver:8080/execution/
    volumes: *airflow-common-volumes
    command: airflow scheduler
    networks:
      - platform_network
    restart: always

  airflow-dag-processor:
    image: airflow-spark-iceberg-teaching:3.2.2-python3.11
    pull_policy: never
    container_name: airflow_dag_processor
    depends_on:
      airflow-init:
        condition: service_completed_successfully
      airflow-webserver:
        condition: service_healthy
    environment:
      <<: *airflow-common-env
      AIRFLOW__CORE__EXECUTION_API_SERVER_URL: http://airflow-webserver:8080/execution/
      AIRFLOW__DAG_PROCESSOR__REFRESH_INTERVAL: "5"
    volumes: *airflow-common-volumes
    command: airflow dag-processor
    networks:
      - platform_network
    restart: always

  mock-lambda-watcher:
    image: python:3.11-slim
    container_name: mock_lambda_watcher
    depends_on:
      airflow-webserver:
        condition: service_healthy
    volumes:
      - ./sftp_mock_daemon.py:/app/sftp_mock_daemon.py:ro
      - ./data:/opt/airflow/data:rw
    working_dir: /app
    environment:
      AIRFLOW_BASE_URL: http://airflow-webserver:8080
      AIRFLOW_USERNAME: admin
      AIRFLOW_PASSWORD: admin
      WATCH_DIRECTORY: /opt/airflow/data/incoming
      TARGET_ASSET_URI: file:///opt/airflow/data/incoming/taxi_data_drop
    networks:
      - platform_network
    # command: >
    #   sh -c "pip install --no-cache-dir watchdog requests &&
    #          python /app/sftp_mock_daemon.py"
    command: >
      sh -c "pip install --no-cache-dir watchdog requests &&
             tail -f /dev/null"
    restart: always

  jupyter-notebook:
    image: airflow-spark-iceberg-teaching:3.2.2-python3.11
    pull_policy: never
    container_name: jupyter_notebook
    depends_on:
      airflow-init:
        condition: service_completed_successfully
    environment:
      <<: *airflow-common-env
      JUPYTER_TOKEN: "dev_token"
    ports:
      - "8888:8888"
    volumes: *airflow-common-volumes
    working_dir: /opt/airflow
    entrypoint: []
    command: >
      jupyter notebook --ip=0.0.0.0 --port=8888 --no-browser --allow-root
    networks:
      - platform_network
    restart: unless-stopped

volumes:
  db_data:

networks:
  platform_network:
    driver: bridge
```


## Test Dag

`./dags/hello_world_dag.py`

```python
from datetime import datetime
import logging

from airflow.sdk import dag, task


@dag(
    dag_id="01_hello_world",
    start_date=datetime(2026, 6, 1),
    schedule=None,
    catchup=False,
    tags=["basics", "infrastructure"],
)
def hello_world_dag():

    @task
    def say_hello():
        log = logging.getLogger("airflow.task")
        log.info("👋 Hello, Airflow 3.x Engine!")

    @task
    def say_goodbye():
        log = logging.getLogger("airflow.task")
        log.info("🏃‍♂️ Logging out of the task container.")

    say_hello() >> say_goodbye()


hello_world_dag()
```


## Test Notebook/Python


```python
import os
import sys
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
import duckdb

print("Python:", sys.version)
print("pandas:", pd.__version__)
print("pyarrow:", pa.__version__)
print("duckdb:", duckdb.__version__)

print("JAVA_HOME:", os.environ.get("JAVA_HOME"))
print("SPARK_HOME:", os.environ.get("SPARK_HOME"))
print("ICEBERG_SPARK_JAR:", os.environ.get("ICEBERG_SPARK_JAR"))

import pyspark

iceberg_jar = os.environ.get("ICEBERG_SPARK_JAR")

print("PySpark:", pyspark.__version__)
print("Iceberg JAR exists:", os.path.exists(iceberg_jar) if iceberg_jar else False)
print("Iceberg JAR path:", iceberg_jar)

```
