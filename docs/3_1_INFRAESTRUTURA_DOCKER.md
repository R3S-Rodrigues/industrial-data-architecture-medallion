# 3.1. Emulação do Chão de Fábrica (Ambiente Local)

Para validar o pipeline sem depender de maquinário físico real, estruturamos uma infraestrutura local conteinerizada utilizando **Docker Compose**. O ambiente simula um Broker MQTT e um script em Python que emite telemetrias industriais continuamente.

### Configuração do `docker-compose.yml`
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

### Arquivo `docs/3_2_GATEWAY_INDUSTRIAL.md`
```markdown
# 3.2. Configuração do Gateway Industrial (OT/IT Convergence)

Esta etapa descreve a arquitetura de mapeamento na borda. O objetivo é estabelecer a comunicação com a árvore de tags do servidor industrial e encapsular as variáveis físicas em mensagens estruturadas para o broker de mensageria.

### A. Configuração das Tags no Kepware Server
1. **Channel Creation:** Criar um canal utilizando o driver nativo correspondente ao CLP (ex: *Siemens TCP/IP Ethernet*).
2. **Device Mapping:** Registrar o ativo com o IP lógico da rede industrial (ex: `192.168.10.24`).
3. **Tag Group & Address:** Estruturar as tags de telemetria cruciais para o cálculo de OEE:
   * `Luanda.Injetora04.Status` (Endereço: `DB1.DBX0.0` - Boolean)
   * `Luanda.Injetora04.Temp_Celsius` (Endereço: `DB1.DBD2` - Real)
   * `Luanda.Injetora04.Total_Counter` (Endereço: `DB1.DBD6` - DInt)
   * `Luanda.Injetora04.Defect_Counter` (Endereço: `DB1.DBD10` - DInt)

###  B. Fluxo de Ingestão e Tradução no Node-RED
Adicione o pacote `node-red-contrib-opcua` e configure o fluxo:
* **Nó OPCUA-Client:** Aponta para `opc.tcp://192.168.10.100:49320` monitorando os NodeIds das tags acima.
* **Nó Function:** Insira o JavaScript para unificar as tags em um payload estruturado:
```javascript
let metrics = context.get('metrics') || { temp_celsius: 0, status: "OFF", total_counter: 0, defect_counter: 0 };

if (msg.topic.includes("Temp_Celsius")) metrics.temp_celsius = parseFloat(msg.payload);
if (msg.topic.includes("Status")) metrics.status = msg.payload ? "RUNNING" : "STOPPED";
if (msg.topic.includes("Total_Counter")) metrics.total_counter = parseInt(msg.payload);
if (msg.topic.includes("Defect_Counter")) metrics.defect_counter = parseInt(msg.payload);

context.set('metrics', metrics);

msg.payload = {
    "asset_id": "CLP_INJ_04_LUANDA",
    "timestamp": new Date().toISOString(),
    "metrics": metrics
};

msg.topic = "factory/luanda/injection/clp_04";
return msg;


### 📄 Arquivo `docs/3_3_PIPELINE_PYSPARK.md`
```markdown
# 3.3. Script do Ingestion Gateway (PySpark)

O processamento contínuo é realizado através de um job PySpark (executado no Databricks ou cluster local). Ele lê o fluxo bruto da Bronze, higieniza-o e persiste em Delta Lake na camada Silver.

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_json
from pyspark.sql.types import StructType, StructField, StringType, DoubleType, IntegerType

spark = SparkSession.builder \
    .appName("Industrial-Data-Ingestion-Silver") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .getOrCreate()

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

df_bronze = spark.readStream.format("delta").table("industrial_db.bronze_raw_events")

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
    .filter(col("machine_name").isNotNull())

query_silver = df_silver.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", "/mnt/checkpoints/industrial_silver") \
    .toTable("industrial_db.silver_trusted_telemetry")


### Arquivo `docs/3_4_CALCULO_OEE.md`
```markdown
# 3.4. Engenharia de Agregados e Cálculo de OEE (Camada Gold)

Para disponibilizar os dados prontos ao Power BI na camada **Gold**, aplicamos as regras matemáticas padrão de manufatura para consolidação do **OEE**.

### Formulação Matemática dos Indicadores
* **Disponibilidade:** $\text{Tempo em Operação} \div \text{Tempo Planejado de Produção}$
* **Performance:** $\text{Produção Real} \div \text{Capacidade Nominal Esperada}$
* **Qualidade:** $\text{Total de Peças Boas} \div \text{Total de Peças Produzidas}$
* **OEE Global:** $\text{Disponibilidade} \times \text{Performance} \times \text{Qualidade}$

### Consulta SQL para Consolidação Analítica (Gold)
```sql
CREATE OR REPLACE TABLE industrial_db.gold_factory_oee_analytics AS
WITH metrics_base AS (
    SELECT 
        DATE(event_timestamp) AS date_day,
        machine_name,
        COUNT(CASE WHEN status = 'RUNNING' THEN 1 END) AS active_minutes,
        480.0 AS planned_production_minutes,
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
    ROUND((active_minutes / planned_production_minutes) * 100, 2) AS availability_pct,
    ROUND((total_produced_in_shift / (active_minutes * 30.0)) * 100, 2) AS performance_pct,
    ROUND(((total_produced_in_shift - total_defects_in_shift) / total_produced_in_shift) * 100, 2) AS quality_pct,
    ROUND(((active_minutes / planned_production_minutes) * (total_produced_in_shift / (active_minutes * 30.0)) * ((total_produced_in_shift - total_defects_in_shift) / total_produced_in_shift)) * 100, 2) AS oee_global_pct
FROM metrics_base;
