# 3. Roteiro Prático de Implementação (Passo a Passo)

Este documento fornece um guia técnico orientado à reprodução do ambiente de engenharia, descrevendo desde a simulação dos ativos industriais via Docker até a computação distribuída e modelagem de métricas de eficiência no Data Lakehouse.

---

## 3.1. Emulação do Chão de Fábrica (Ambiente Local)

Para validar o pipeline sem depender de maquinário físico real, estruturamos uma infraestrutura local conteinerizada utilizando **Docker Compose**. O ambiente simula um Broker MQTT e um script em Python que emite telemetrias industriais continuamente.

### 📄 Configuração do `docker-compose.yml`
```yaml
version: '3.8'

services:
  # Broker MQTT para recepção dos eventos da borda
  mqtt-broker:
    image: eclipse-mosquitto:2.0
    container_name: industrial_mqtt_broker
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
    restart: always

  # Simulador que consome a árvore de tags e envia payloads JSON
  factory-simulator:
    image: python:3.10-slim
    container_name: factory_edge_simulator
    depends_on:
      - mqtt-broker
    volumes:
      - ./simulator:/app
    command: sh -c "pip install paho-mqtt && python /app/simulator.py"
    restart: always


# Script do Ingestion Gateway (PySpark)

from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_json, current_timestamp
from pyspark.sql.types import StructType, StructField, StringType, DoubleType, IntegerType

# 1. Inicialização da Sessão Spark com suporte ao formato Delta
spark = SparkSession.builder \
    .appName("Industrial-Data-Ingestion-Silver") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

# 2. Definição do Esquema estrito para decodificação do payload JSON industrial
schema_ot = StructType([
    StructField("asset_id", StringType(), True),
    StructField("timestamp", StringType(), True),
    StructField("metrics", StructType([
        StructField("temp_celsius", DoubleType(), True),
        StructField("status", StringType(), True),
        StructField("total_counter", IntegerType(), True),
        StructField("defect_counter", IntegerType(), True)
    ]), True)
])

# 3. Leitura dos dados brutos a partir da Camada Bronze (Delta Lake)
df_bronze = spark.readStream.format("delta").table("industrial_db.bronze_raw_events")

# 4. Processamento, limpeza e desaninhamento dos dados brutos
df_silver = df_bronze \
    .withColumn("parsed_json", from_json(col("raw_payload"), schema_ot)) \
    .select(
        col("parsed_json.timestamp").cast("timestamp").alias("event_timestamp"),
        col("parsed_json.asset_id").alias("machine_name"),
        col("parsed_json.metrics.status").alias("status"),
        col("parsed_json.metrics.temp_celsius").alias("temperature"),
        col("parsed_json.metrics.total_counter").alias("total_produced"),
        col("parsed_json.metrics.defect_counter").alias("defective_units")
    ) \
    .filter(col("machine_name").isNotNull()) # Regra de qualidade básica

# 5. Escrita contínua e persistência na tabela confiável (Camada Silver)
query_silver = df_silver.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", "/mnt/checkpoints/industrial_silver") \
    .toTable("industrial_db.silver_trusted_telemetry")


# Consulta SQL para Consolidação Analítica (Gold)

CREATE OR REPLACE TABLE industrial_db.gold_factory_oee_analytics AS
WITH metrics_base AS (
    SELECT 
        DATE(event_timestamp) AS date_day,
        machine_name,
        -- Mapeamento de tempos operacionais simulados (em minutos)
        COUNT(CASE WHEN status = 'RUNNING' THEN 1 END) AS active_minutes,
        480.0 AS planned_production_minutes, -- Equivalente a 1 turno padrão
        MAX(total_produced) - MIN(total_produced) AS total_produced_in_shift,
        MAX(defective_units) - MIN(defective_units) AS total_defects_in_shift,
        AVG(temperature) AS avg_temperature
    FROM industrial_db.silver_trusted_telemetry
    GROUP BY DATE(event_timestamp), machine_name
)
SELECT 
    date_day,
    machine_name,
    avg_temperature,
    
    -- 1. Cálculo de Disponibilidade
    ROUND((active_minutes / planned_production_minutes) * 100, 2) AS availability_pct,
    
    -- 2. Cálculo de Performance (Assumindo meta nominal de 30 peças por minuto)
    ROUND((total_produced_in_shift / (active_minutes * 30.0)) * 100, 2) AS performance_pct,
    
    -- 3. Cálculo de Qualidade
    ROUND(((total_produced_in_shift - total_defects_in_shift) / total_produced_in_shift) * 100, 2) AS quality_pct,
    
    -- 4. Cálculo do OEE Global
    ROUND(
        ((active_minutes / planned_production_minutes) * (total_produced_in_shift / (active_minutes * 30.0)) * ((total_produced_in_shift - total_defects_in_shift) / total_produced_in_shift)) * 100, 2
    ) AS oee_global_pct
FROM metrics_base;
