# 3.1. Conectividade Industrial Edge: Kepware KepServerEX & Gateway MQTT

Esta etapa detalha a camada de aquisição de dados na borda (Edge), mapeando o fluxo desde os ativos físicos do chão de fábrica até o Gateway Industrial. O objetivo principal é a conversão dos protocolos nativos industriais (Modbus TCP, Siemens S7, Ethernet/IP) para um padrão de alta eficiência e leveza para redes de TI: o protocolo **MQTT (Message Queuing Telemetry Transport)** via **JSON Payload**.

---

## 3.1.1. Modelagem e Estruturação de Tags no Kepware

Para garantir a padronização e a governança dos dados antes mesmo do envio à nuvem, foi estabelecida uma topologia de nomenclatura hierárquica baseada na norma **ISA-95** (`Planta_Setor_Ativo_Componente`):

* **Channel (Canal de Comunicação):** `Linha_Producao_01` (Driver: *Siemens TCP/IP Ethernet*)
* **Device (Dispositivo/PLC):** `PLC_Principal_S71200`
* **Tags Cadastradas:**

| Nome da Tag (Tag Name) | Endereço Físico (Address) | Tipo de Dado | Descrição |
| :--- | :--- | :--- | :--- |
| `Sensor_Temperatura_Mancal` | `DB1.DBD0` | Real (Float) | Temperatura interna do mancal principal da bomba (°C) |
| `Status_Operacional` | `DB1.DBX4.0` | Boolean | Estado de operação (0: Parado, 1: Em Operação) |
| `Contador_Pulsos_Producao` | `DB1.DBW6` | Word (Int) | Totalizador de unidades processadas no turno |
| `Alarme_Pressao_Critica` | `DB1.DBX8.0` | Boolean | Indicador de sobrepressão na tubulação de recalque |

---

## 3.1.2. Configuração do MQTT IoT Gateway (Conversão OPC UA -> MQTT)

Dentro do ecossistema do **KEPServerEX**, utilizamos o plug-in **IoT Gateway** atuando como um **MQTT Client**. A parametrização foi projetada para otimizar o consumo de banda através de envios orientados a eventos (*Report by Exception - RBE*).

### Parâmetros de Conexão do Gateway:
* **MQTT Broker URL:** `ssl://broker.industrial-it.local:8883` (Comunicação criptografada TLS v1.3)
* **Client ID:** `Edge_Gateway_Luanda_Plant01`
* **Quality of Service (QoS):** `QoS 1 (At least once)` - Garante a entrega da telemetria fabril sem perdas.
* **Topic Structure:** `industria/luanda/linha01/telemetria`
* **Data Scan Rate (Taxa de Varredura):** `1000ms`
* **Trigger (Gatilho de Transmissão):** `On Data Change` (Deadband de 0.5% para variáveis analógicas, reduzindo ruídos de sensores).

---

## 3.1.3. Payload de Transmissão (Modelo JSON Padronizado)

Abaixo está o exemplo real do payload gerado pelo IoT Gateway do Kepware e injetado no broker para consumo da arquitetura Medallion. Cada tag é enviada contendo seu valor, timestamp com precisão de milissegundos e a flag de qualidade OPC UA (`Good = true`):

```json
{
  "timestamp": "2026-07-11T16:36:00.124Z",
  "plant_location": "Luanda_Icolo_Bengo",
  "asset_id": "PLC_Principal_S71200",
  "readings": [
    {
      "tag": "Linha_Producao_01.PLC_Principal_S71200.Sensor_Temperatura_Mancal",
      "value": 74.85,
      "quality": "Good"
    },
    {
      "tag": "Linha_Producao_01.PLC_Principal_S71200.Status_Operacional",
      "value": true,
      "quality": "Good"
    },
    {
      "tag": "Linha_Producao_01.PLC_Principal_S71200.Contador_Pulsos_Producao",
      "value": 1425,
      "quality": "Good"
    },
    {
      "tag": "Linha_Producao_01.PLC_Principal_S71200.Alarme_Pressao_Critica",
      "value": false,
      "quality": "Good"
    }
  ]
}



## 3.2. Diagrama de Arquitetura da Borda
![Conectividade Industrial Edge](./Arquitetura_Ingestão_Edge_Kepware.png)

# 3.2.1. Conectividade Industrial Edge: Kepware KEPServerEX & Gateway MQTT para Azure Databricks

Esta etapa detalha a camada de aquisição de dados na borda (Edge), mapeando o fluxo completo desde os ativos físicos do chão de fábrica até a conversão e direcionamento seguro para processamento analítico em nuvem.

---

## 3.2.2. Diagrama de Arquitetura da Borda (Conectividade Industrial)

Abaixo está a representação visual do ecossistema de dados unificado, demonstrando o fluxo contínuo de conversão OT/TI:

![Conectividade Industrial Edge](./Conectividade_Industrial_Edge_KEPServerEX.png)

---

## 3.2.3. Mapeamento dos Componentes do Pipeline

Com base na arquitetura desenhada, o fluxo de dados é segregado em três grandes domínios operacionais:

### 1. Chão de Fábrica (Fontes de Dados)
A camada operacional é composta por CLPs, sensores IoT e sistemas industriais SCADA de múltiplos protocolos nativos:
* **Protocolos Suportados:** Modbus TCP, Siemens S7 e Ethernet/IP.
* **Padronização de Tags (Norma ISA-95):** Para garantir consistência semântica e governança, os pontos de telemetria utilizam a topologia hierárquica `Planta_Setor_Ativo_Componente`. Exemplos de medições mapeadas:
  * `Sensor_Temperatura_Mancal`
  * `Status_Operacional`
  * `Contador_Pulsos_Producao`
  * `Alarme_Pressao_Critica`

### 2. KEPWARE KEPServerEX (Camada de Conversão Borda/Edge)
O software atua como a engine de interoperabilidade centralizada na borda, dividida em dois núcleos funcionais:
* **OPC UA Server (Norma IEC 62541):** Concentra a comunicação vinda dos dispositivos de automação através de canais criptografados com Broker TLS Seguro, fornecendo resiliência local por meio de mecanismos de **Store & Forward** (para bufferização em caso de instabilidade na rede).
* **IoT Gateway MQTT:** Executa a efetiva **Conversão OT / MQTT**, encapsulando as variáveis industriais em payloads leves formato JSON de alta eficiência.

### 3. Cloud: Azure Databricks (Recepção e Processamento)
Os dados empacotados na borda são transmitidos via conexão cifrada segura (MQTTS) para o ambiente de nuvem, habilitando as seguintes capacidades no Lakehouse:
* **Recepção Segura:** Endpoints expostos de forma controlada com autenticação rigorosa.
* **Processamento em Tempo Real:** Ingestão contínua em buffers de streaming para tratamento imediato da telemetria.
* **Análise Escalável:** Disponibilização dos conjuntos de dados limpos para algoritmos de engenharia, dashboards analíticos e modelos preditivos.

