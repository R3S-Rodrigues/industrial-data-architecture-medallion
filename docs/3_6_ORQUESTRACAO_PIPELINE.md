
# 3.6. Orquestração de Pipelines no Databricks Workflows (Jobs)

Este documento descreve a configuração visual da orquestração de ponta a ponta da arquitetura Medallion no Databricks. Utilizaremos o **Databricks Workflows (Jobs)** para encadear a ingestão (Bronze) e o refinamento de séries temporais (Prata), garantindo governança, automação e monitoramento de falhas.

---

## 3.6.1. Design do Grafo de Dependências (DAG)

O fluxo operacional foi desenhado para rodar de forma incremental e ordenada:

## 3.6.2. Passo a Passo para Configuração na Interface Visual

Siga as etapas abaixo na tela de **Jobs e Pipelines** do Databricks para construir a automação:

### Passo 1: Criar o Job Principal
1. Na barra superior direita da sua tela, clique no botão azul **Criar** (ou selecione o card **Job**).
2. No canto superior esquerdo da nova tela, altere o nome padrão do Job de `Untitled Job` para:  
   `job_oil_gas_medallion_pipeline`

### Passo 2: Configurar a Primeira Tarefa (Camada Bronze)
Na janela de configuração da primeira tarefa (*Task*), defina os seguintes parâmetros:
* **Task name:** `Ingestao_Bronze`
* **Type:** `Notebook`
* **Source:** `Workspace`
* **Path:** Selecione o seu notebook `ingestao_oil_gas_bronze`
* **Compute (Cluster):** Escolha o seu cluster ativo (ou configure um cluster de Job para otimização de custo).
* Clique em **Create task**.

### Passo 3: Adicionar a Segunda Tarefa (Camada Prata)
1. No painel central (onde agora exibe o grafo visual), passe o mouse sobre a tarefa `Ingestao_Bronze` e clique no ícone de **`+` (Add task)** logo abaixo dela.
2. Configure a nova tarefa com os seguintes parâmetros:
   * **Task name:** `Refino_Silver_TimeSeries`
   * **Type:** `Notebook`
   * **Path:** Selecione o seu notebook de transformação Silver (`pyspark_oil_gas_silver`)
   * **Depends on:** Garanta que esta opção esteja marcada como `Ingestao_Bronze` (isso cria a linha de conexão visual entre as duas tarefas).
3. Clique em **Create task**.

---

## 3.6.3. Agendamento e Execução (*Trigger*)

Com o grafo visual montado, você pode definir como o pipeline será disparado no painel lateral direito (**Job details**):

1. **Execução Manual (Testes):** Clique no botão **Run now** no canto superior direito para disparar o pipeline completo imediatamente. O Databricks abrirá uma janela de rastreamento para você assistir a execução de cada notebook em tempo real.
2. **Execução Agendada (Produção):** Na seção **Triggers**, clique em **Add trigger**:
   * **Trigger type:** `Scheduled` (Cron)
   * **Schedule:** Defina a periodicidade industrial necessária (ex: Todos os dias às 02:00 AM, ou a cada hora, dependendo do volume de arquivos no seu *Volume Gerenciado*).
  
   * 
