
# 3.5. Desenvolvimento do Pipeline da Camada Bronze (Auto Loader & Delta Lake)

Esta etapa documenta o desenvolvimento do pipeline de ingestão contínua responsável por capturar os arquivos brutos depositados no ecossistema do Data Lake, salvando-os de forma transacional na camada **Bronze** em formato **Delta Lake**. O design foca na captura rápida, imutabilidade dos dados de entrada e preservação das características originais das fontes de Oil & Gas.

---

## 3.5.1. Estrutura do DDL de Inicialização (SQL)

Antes de iniciar a ingestão via streaming incremental, estruturamos os objetos lógicos de governança no **Unity Catalog** (`oil_gas_catalog`) e ativamos políticas avançadas de gerenciamento de colunas.

```sql
-- 1. Criar o Catálogo Principal para o projeto de Oil & Gas
CREATE CATALOG IF NOT EXISTS oil_gas_catalog;

-- 2. Definir o catálogo criado como o contexto atual
USE CATALOG oil_gas_catalog;

-- 3. Criar o Esquema Bronze dentro do Catálogo
CREATE SCHEMA IF NOT EXISTS bronze;

-- 4. Criar o Volume Gerenciado para armazenar o arquivo CSV bruto
CREATE VOLUME IF NOT EXISTS bronze.raw_files;


# CONFIGURAR E EXECUTAR A CELULA DE INGESTÃO

# 1. Definição dos caminhos de armazenamento dentro do Unity Catalog
volume_path = "/Volumes/oil_gas_catalog/bronze/raw_files/"
checkpoint_path = "/Volumes/oil_gas_catalog/bronze/raw_files/_checkpoints/oil_gas_production"
target_table = "oil_gas_catalog.bronze.producao_bruta"

# 2. Criar a tabela com Column Mapping habilitado antes da escrita streaming
spark.sql(f"""
    CREATE TABLE IF NOT EXISTS {target_table}
    TBLPROPERTIES ('delta.columnMapping.mode' = 'name')
""")

# 3. Configuração do Databricks Auto Loader para ler o CSV automaticamente
df_auto_loader = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("cloudFiles.schemaLocation", checkpoint_path)  # Gerencia a evolução do esquema
    .option("delimiter", ";")                               # Arquivo delimitado por ponto e vírgula
    .option("header", "true")                               # Primeira linha contém os cabeçalhos
    .option("inferSchema", "true")                          # Detecta tipos de dados iniciais
    .load(volume_path)
)

# 4. Escrita incremental e contínua para a Tabela Delta na Camada Bronze
query = (df_auto_loader.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", checkpoint_path)
    .option("mergeSchema", "true")
    .trigger(availableNow=True) # Processa todos os dados disponíveis e encerra o micro-batch
    .toTable(target_table)
)


-- Amostragem analítica para validação dos dados da ANP na Camada Bronze
SELECT * FROM oil_gas_catalog.bronze.producao_bruta LIMIT 5;

