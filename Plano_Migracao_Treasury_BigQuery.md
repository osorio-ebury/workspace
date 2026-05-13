# Documentação de Migração: Conexão ODBC BigQuery - Time de Tesouraria

## 1. Visão Geral
Este documento detalha o plano de migração e a configuração técnica da conexão ODBC utilizada pela planilha da Tesouraria. O objetivo é garantir a transição da infraestrutura de dados para um novo projeto no Google Cloud Platform (GCP) sem impactar as operações diárias.

**Identificação do Card:** EPT-2075

---

## 2. Estado Atual (Legado)
Atualmente, o time de tesouraria utiliza uma planilha que consome dados diretamente do BigQuery para suporte à tomada de decisões.

* **Projeto GCP:** `fx-production`
* **Dataset:** `treasury`
* **Tabela/View:** `treasury_vw2`
* **Driver Utilizado:** Simba ODBC Driver for Google BigQuery (64-bit).
* **Mecanismo de Autenticação:** OAuth (User Authentication).
* **Ponto de Atualização:** Serviço de FX externo.

---

## 3. Plano de Migração

### 3.1. Estratégia de Acesso (IAM)
Para aumentar a segurança e facilitar a gestão, a migração seguirá as seguintes premissas:
1.  **Criação de Grupo:** Todos os usuários da tesouraria serão adicionados a um grupo Google (ex: `gcp-treasury-users@dominio.com`).
2.  **Custom Role:** Será criada uma Role Customizada contendo apenas o estritamente necessário para a leitura e execução de jobs.

#### Permissões Necessárias (Custom Role):
Com base na análise técnica, as permissões mínimas são:
* `bigquery.datasets.get`
* `bigquery.datasets.getIamPolicy`
* `bigquery.jobs.create` (Necessário para executar as consultas)
* `bigquery.readsessions.create`
* `bigquery.readsessions.getData`
* `bigquery.tables.list`
* `resourcemanager.projects.get`

### 3.2. Procedimento de Configuração (DSN)
A alteração será realizada no **ODBC Data Source Administrator (64-bit)** nas estações de trabalho/servidores de suporte.

**Passos para o Suporte (Responsável: Chrystian):**
1.  Abrir o **ODBC Data Source Administrator**.
2.  Na aba **System DSN**, selecionar a fonte `Google BigQuery` e clicar em **Configure**.
3.  No campo **Catalog (Project)**, alterar de `fx-production` para o **[NOVO_PROJETO_GCP]**.
4.  No campo **Dataset**, validar ou alterar para o novo nome do dataset correspondente.
5.  Clicar em **Test** para validar a conectividade.
6.  Clicar em **OK** para salvar.

---

## 5. Referências e Recursos
* **Driver ODBC:** [Download Simba ODBC Driver](https://docs.cloud.google.com/bigquery/docs/reference/odbc-jdbc-drivers?hl=pt-br)
* **Pessoas Chave:**
    * **Negócio/Processos:** Alessandro Giglio e Flávio Santos.
    * **Execução Técnica/Suporte:** Chrystian.
* **Usuários com acesso à planilha:**
    * Eduardo Moutinho
    * Henrique Huller
    * Marco Rigo