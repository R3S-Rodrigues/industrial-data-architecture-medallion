
# 3.1.1. Arquitetura de Ingestão Edge com Kepware KEPServerEX

Esta subetapa documenta o mapeamento técnico e a representação visual da arquitetura de conectividade industrial na borda (Edge), estabelecendo as diretrizes para a convergência OT/TI estável, segura e escalável.

---

## 3.1.1.1. Diagrama de Arquitetura da Borda (Conectividade Industrial)

Abaixo está o ecossistema unificado, demonstrando o fluxo contínuo de conversão de protocolos industriais nativos para mensageria MQTT leve, direcionada ao Lakehouse:

![Conectividade Industrial Edge](./Arquitetura_Ingestão_Edge_Kepware.png)

---

## 3.1.1.2. Mapeamento Estrutural dos Domínios Operacionais

Com base no modelo de arquitetura acima, o fluxo de dados divide-se em três pilares fundamentais:

### 1. Chão de Fábrica (Camada OT - Fontes de Dados)
Composta por ativos e instrumentação industrial de múltiplos fornecedores e protocolos legados:
* **Protocolos e Sistemas:** Integração de CLPs, Sensores IoT e Sistemas SCADA baseados em Modbus TCP, Siemens S7 e Ethernet/IP.
* **Governança de Tags (Norma ISA-95):** Padronização semântica utilizando a hierarquia estruturada `Planta_Setor_Ativo_Componente`. Exemplos reais:
  * `Sensor_Temperatura_Mancal`
  * `Status_Operacional`
  * `Contador_Pulsos_Producao`
  * `Alarme_Pressao_Critica`

### 2. Camada Edge (KEPServerEX e Interoperabilidade)
Plataforma central de tradução e resiliência de dados na borda da fábrica:
* **OPC UA Server (Norma IEC 62541):** Concentra o tráfego dos drivers industriais em canais TLS seguros e criptografados, aplicando políticas nativas de **Store & Forward** para reter dados localmente no caso de quedas de link de rede.
* **IoT Gateway MQTT:** Realiza a efetiva **Conversão OT / MQTT**, serializando os estados físicos das tags em objetos JSON estruturados de baixo overhead.

### 3. Camada Cloud (Azure Databricks Lakehouse)
Ponto de recepção e processamento massivo em nuvem:
* **Recepção e Segurança (MQTTS):** Terminação de endpoints protegidos por certificados de segurança robustos.
* **Ingestão Streaming:** Captura contínua de telemetria em tempo real diretamente para as camadas de processamento.
* **Escalabilidade Computacional:** Disponibilização dos vetores limpos para pipelines analíticos avançados e modelos preditivos de machine learning.
