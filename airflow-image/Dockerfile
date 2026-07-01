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