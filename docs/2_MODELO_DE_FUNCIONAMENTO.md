# 2. Modelo de Funcionamento (Arquitetura e Fluxo de Dados)

Este documento descreve detalhadamente o fluxo contínuo dos dados industriais (*end-to-end*), mapeando a jornada desde a geração do sinal físico nos sensores do chão de fábrica até a persistência estruturada e consumo analítico nas camadas do Data Lakehouse.

---

## 2.1. Arquitetura de Fluxo de Dados (Jornada do Dado)

O pipeline foi projetado para operar com base em eventos em tempo real, mitigando o acoplamento entre os sistemas de automação (OT) e os sistemas analíticos corporativos (IT). O fluxo segue o ciclo:

1. **Geração do Dado (Borda / OT):** Sensores físicos medem grandezas (Temperatura, Contagem de Peças, Pressão) e comunicam-se com os CLPs (Controladores Lógicos Programáveis) através de redes industriais de campo.
2. **Centralização Local (Servidor OPC UA):** Um servidor de automação (ex: Kepware ou Siemens Telecontrol) centraliza as tags operacionais em uma árvore de objetos padronizada e segura.
3. **Ingestão e Adaptação (IoT Gateway):** Um agente leve na borda (Edge Gateway, como Node-RED) subscreve-se às tags críticas do servidor OPC UA. Ele atua como um tradutor de protocolo, convertendo os dados binários ou estruturas industriais rígidas em mensagens textuais compactadas no formato JSON padrão.
4. **Transporte e Mensageria (Nuvem / IT Hub):** O Gateway despacha as mensagens JSON de telemetria em alta velocidade via protocolo MQTT para um broker de mensageria escalável (ex: Azure IoT Hub ou Apache Kafka).
5. **Processamento Distribuído (Lakehouse):** Um motor de computação distribuída (PySpark/Databricks) consome os tópicos continuamente e distribui a carga estrutural pelas camadas da arquitetura Medallion.

---

## 2.2. Divisão de Responsabilidades da Arquitetura Medallion

A organização dos dados no Data Lakehouse utiliza o padrão de **Arquitetura Medallion**, dividida em três zonas lógicas que garantem qualidade, governança e rastreabilidade:

### 🟫 Camada Bronze (Dados Brutos / Raw Data)
* **Objetivo:** Persistir os dados exatamente como foram recebidos das fontes emissoras da fábrica, sem alterações de valor ou tipo.
* **Comportamento:** Funciona como uma tabela histórica imutável (*append-only*). Se houver uma falha de conversão posterior, o dado bruto original está seguro para reprocessamento.
* **Estrutura:** Mantém os payloads no formato JSON original dentro de colunas de texto (string), acompanhados por metadados de ingestão (data/hora de recebimento no servidor e partição por data).

### ⬜ Camada Silver (Dados Estruturados e Limpos / Trusted Data)
* **Objetivo:** Fornecer uma visão limpa, tipada, enriquecida e semanticamente consistente dos dados operacionais da planta.
* **Comportamento:** Aplica regras de qualidade rigorosas sobre a tabela Bronze. É a camada ideal para auditoria técnica de engenharia de processos e respostas rápidas do Centro de Controle Operacional (CCO).
* **Transformações Aplicadas:**
  * Extração e quebra dos campos do JSON em colunas fortemente tipadas (Integer, Double, Timestamp).
  * Remoção de registros duplicados e tratamento de anomalias (ex: leituras nulas causadas por perda pontual de sinal).
  * Enriquecimento contextual (*Lookup*): União da telemetria com tabelas de dimensões corporativas (ex: cruzando o `Equipment_ID` com tabelas cadastrais que informam o Nome da Máquina, Linha de Produção, Setor e Fábrica).

### 🟨 Camada Gold (Dados Analíticos e KPIs / Refined Data)
* **Objetivo:** Disponibilizar tabelas altamente otimizadas, agregadas e modeladas para atender diretamente aos casos de uso de negócios, dashboards executivos e modelos preditivos.
* **Comportamento:** Armazena dados agregados em janelas temporais de produção (por turno, por hora ou por dia). Consome diretamente da camada Silver e organiza as tabelas no formato dimensional (Fatos e Dimensões / Star Schema).
* **Métricas Calculadas:** Tabelas prontas com cálculo consolidado de Eficiência Global de Equipamentos (**OEE**), segmentadas por:
  * **Disponibilidade:** Tempo de máquina operando vs. tempo planejado.
  * **Performance:** Velocidade real de produção vs. capacidade nominal do ativo.
  * **Qualidade:** Peças boas produzidas vs. total de peças injetadas/manufaturadas.

---

## 2.3. Exemplo Prático de Evolução do Dado (Payloads)

Para ilustrar o funcionamento contínuo do pipeline, veja abaixo como um único evento de telemetria evolui estruturalmente ao longo das camadas do ecossistema:

### 📥 1. Payload de Ingestão na Borda (JSON enviado pelo Gateway)
```json
{
  "asset_id": "CLP_INJ_04_LUANDA",
  "timestamp": "2026-07-09T14:30:00Z",
  "metrics": {
    "temp_celsius": 185.4,
    "status": "RUNNING",
    "total_counter": 12450,
    "defect_counter": 120
  }
