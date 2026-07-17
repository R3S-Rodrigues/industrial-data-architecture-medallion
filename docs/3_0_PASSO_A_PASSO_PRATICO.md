# 3. Roteiro Prático de Implementação (End-to-End)

Este módulo serve como o mapa de execução do pipeline prático. Para facilitar o deploy, o isolamento de testes e a manutenção da arquitetura, o roteiro foi modularizado em subdiretórios que cobrem a jornada completa do dado, da automação (OT) à inteligência de negócios (IT).

Siga as etapas na ordem sequencial abaixo para reproduzir o ecossistema de dados industriais:

---

###  [Etapa 3.1: Emulação da Infraestrutura Local (Docker)](./3_1_INFRAESTRUTURA_DOCKER.md)
*Subida automatizada dos serviços base para suportar o fluxo de mensageria.*
* Criação do Broker MQTT local (Mosquitto).
* Script Python de simulação contínua do comportamento mecânico de injetoras.

###  [Etapa 3.2: Configuração do Gateway Industrial (OT/IT)](./3_2_GATEWAY_INDUSTRIAL.md)
*Mapeamento e enquadramento de dados na borda da fábrica.*
* Estruturação de tags no Kepware Server (OPC UA).
* Desenvolvimento de fluxos lógicos e tradução em Node-RED para geração de JSONs.

###  [Etapa 3.3: Pipeline de Ingestão Contínua (PySpark)](./3_3_PIPELINE_PYSPARK.md)
*Consumo e transformação distribuída através da Arquitetura Medallion.*
* Conexão e leitura de streams em tempo real.
* Processamento, limpeza e persistência Delta Lake na Camada Silver.

###  [Etapa 3.4: Engenharia de Agregados e Cálculo do OEE (SQL)](./3_4_CALCULO_OEE.md)
*Modelagem dimensional da camada semântica final voltada à tomada de decisão.*
* Regras matemáticas formais de Disponibilidade, Performance e Qualidade.
* Query analítica avançada para a consolidação da Camada Gold.
