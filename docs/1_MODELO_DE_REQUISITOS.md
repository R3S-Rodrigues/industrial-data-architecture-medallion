# 1. Modelo de Requisitos

Este documento estabelece as especificações, objetivos e restrições que guiam o desenvolvimento da **Arquitetura Unificada de Dados Industriais**. Ele serve como garantia de que a solução técnica atenda diretamente às necessidades operacionais e analíticas do negócio (Indústria 4.0).

---

## 1.1. Identificação
* **Nome do Projeto:** Arquitetura Unificada de Dados Industriais: Convergência OT/IT com Gateway IoT e Data Lakehouse Medallion para Analytics em Tempo Real.
* **Autor:** Eng. Elias Rodrigues U. Zimbeti
* **Função:** Industrial Data Engineer (OT/IT Convergence & Analytics)
* **Contexto:** Indústria 4.0, Internet das Coisas Industrial (IIoT), Big Data e Analytics.

---

## 1.2. Objetivo
O objetivo deste projeto é projetar e implementar um pipeline de dados *end-to-end* robusto, seguro e escalável. A solução realiza a ingestão de dados em tempo real vindos do Chão de Fábrica (Camada OT) via protocolos industriais, estruturando-os em um Data Lakehouse através da **Arquitetura Medallion** para viabilizar:
1. Monitoramento em tempo real e gestão de alarmes críticos em um Centro de Controle Operacional (CCO).
2. Análise de Eficiência Global dos Equipamentos (OEE) por equipes de engenharia de processo.
3. Disponibilização de dados refinados para relatórios executivos (Power BI) e modelos preditivos de Machine Learning.

---

## 1.3. Público-alvo / Usuários
* **Operadores e Supervisores de Planta (CCO):** Necessitam de baixa latência para monitorar o status das máquinas e reagir a alarmes críticos.
* **Engenheiros de Processo e Produção:** Consumidores de dados históricos e agregados para cálculo de KPIs, análise de gargalos e melhoria contínua (OEE).
* **Gestores e Diretores de Operações (C-Level):** Utilizam visões macro e dashboards estratégicos para tomada de decisão financeira e produtiva.
* **Cientistas de Dados / Analistas de BI:** Consumem dados limpos e modelados para criar insights preditivos (ex: manutenção preventiva).

---

## 1.4. Requisitos Funcionais (FR)
Os Requisitos Funcionais descrevem as ações e comportamentos que o sistema deve executar:

| ID | Requisito Funcional | Descrição |
| :--- | :--- | :--- |
| **RF-01** | Coleta de Tags Industriais | O sistema deve se conectar a CLPs, sensores IoT e sistemas SCADA para coletar variáveis físicas (ex: temperatura, pressão, contadores). |
| **RF-02** | Suporte a Protocolos OT | A ingestão na borda (*Edge*) deve suportar nativamente os protocolos padrão de mercado **OPC UA** e **MQTT**. |
| **RF-03** | Ingestão Imutável (Bronze) | O pipeline deve persistir os dados brutos exatamente como foram coletados na camada inicial (**Bronze**), garantindo a rastreabilidade. |
| **RF-04** | Higienização e Contexto (Silver) | O sistema deve limpar os dados brutos (remover duplicatas, tratar nulos) e enriquecê-los com dados de cadastro corporativo na camada (**Silver**). |
| **RF-05** | Agregação de KPIs (Gold) | O sistema deve calcular automaticamente métricas de negócio e o OEE em janelas temporais configuráveis na camada (**Gold**). |
| **RF-06** | Distribuição de Dados | A camada final deve disponibilizar conectores otimizados para consumo direto pelo Power BI e ambientes de Machine Learning. |

---

## 1.5. Requisitos Não Funcionais (NFR)
Os Requisitos Não Funcionais definem as características de qualidade e restrições do sistema:

* **RNF-01 (Disponibilidade e Resiliência):** O Gateway Industrial na borda deve possuir capacidade de *Store-and-Forward*. Em caso de perda de conectividade com a nuvem/data center, os dados devem ser cacheados localmente para evitar perda de histórico.
* **RNF-02 (Desempenho/Latência):** O fluxo de ponta a ponta para alertas críticos direcionados ao CCO deve possuir latência submétrica (próximo ao tempo real). O pipeline analítico geral pode operar em micro-batching ou streaming conforme custo/benefício.
* **RNF-03 (Segurança e Isolamento OT/IT):** O Gateway deve atuar como uma barreira de segurança, garantindo que a rede de automação (OT) fique isolada de acessos diretos vindos da rede corporativa ou internet (IT), utilizando zonas desmilitarizadas (DMZ) se necessário.
* **RNF-04 (Escalabilidade):** A arquitetura do Data Lakehouse deve ser baseada em computação distribuída ou armazenamento em nuvem elástico, permitindo o acréscimo de novas máquinas ou sensores sem degradação de performance.

---

## 1.6. Entradas e Saídas

### Entradas:
* Séries temporais e telemetria de sensores (temperatura, vibração, velocidade).
* Logs de eventos, estados de máquina (Rodando, Parada, Falha) e alarmes gerados por SCADA.
* Tabelas de contexto de sistemas de manufatura (Ex: `dim_equipment`, cadastro de turnos e operadores).

### Saídas:
* Streams de dados em tempo real para telas de monitoramento do CCO.
* Tabelas analíticas de OEE e KPIs consolidadas na camada Gold.
* Datasets estruturados para consumo por modelos de Machine Learning.
* Relatórios visuais interativos e dinâmicos no Power BI.

---

## 1.7. Premissas e Restrições
* **Premissa 01:** Os ativos de chão de fábrica já estão integrados a uma rede industrial comum e comunicam-se de forma centralizada com um servidor OPC UA ou Broker MQTT (ex: via Kepware, Siemens, Ignition ou similar).
* **Restrição 01:** Restrições rígidas de largura de banda na rede industrial impedem o envio de payloads pesados, tornando obrigatório o uso de formatos compactados (como JSON enxuto ou binários) e técnicas de filtragem na borda (*Edge Filtering*).
