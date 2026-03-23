# Documentação Técnica - Relatório Programação de Pagamento (Power BI)
## 1. Visão Geral
Este projeto integra duas frentes de dados:
- BigQuery (DW para planilhas online de Controle Orçamentário Diário +
silver/gold views)
- SQL Server (source de `FACT_CPR_PEDIDO`, `FACT_CPR_SOLICITACAO` e
`STAGING_PEDIDOS_EMETIDOS`)
O objetivo é construir um dataset de análise para a Saritur, com foco em
custo, consumo e planejamento operacional.
## 2. Data Sources
### 2.1 BigQuery (via Power BI e Python ETL)
Project: `projeto-suprimentos-484211`
Schemas/tables utilizados:
- `gold_analytics_us.tbl_pagamentos_consolidados` → alimenta `fDiario`
- `gold_analytics_us.tbl_entrada_emergencial` → alimenta `fEmergencial`
- `gold_analytics_us.tbl_entrada_backolog` → alimenta `fBacklog_Pedido`
- `gold_analytics_us.tbl_entrada_backlog_solic` → alimenta `fBacklog_Solic`
- `silver_cleaned_us.v_planilha_...` (views geradas por scripts Python dos
apps) como `v_planilha_diaria`, `v_planilha_emergencial`,
`v_planilha_backlog`, `v_planilha_backlog_solic`.
### 2.2 SQL Server (via scripts Python)
Conexão: variável de ambiente `SQL_CONNECTION_STRING`.
Query base do `app_*`:
- `FACT_CPR_PEDIDO` + `STAGING_PEDIDOS_EMETIDOS`
(app_diario/app_emergencial/app_backlog)
- `FACT_CPR_SOLICITACAO` (app_backlog_solic)
### 2.3 Arquivos locais (Excel)
- `C:\Users\Anl-suprimentos\Documents\tbl_orcamento.xlsx` (tabela
`tbl_orcamento` no modelo Power BI)
## 3. 4 Apps Python e o fluxo BigQuery + SQL Server
A pasta principal contém 4 scripts:
- `app_diario.py`
- `app_emergencial.py`
- `app_backlog.py`
- `app_backlog_solic.py`
Fluxo padrão por app:
1. `update_silver()` -> cria/atualiza `silver_cleaned_us.v_planilha_*` no
BigQuery com `CREATE OR REPLACE TABLE` a partir de
`bronze_staging_us.src_planilha_*`.
2. `etl_sql()` -> lê dados do SQL Server (`FACT_CPR_*` +
`STAGING_PEDIDOS_EMETIDOS`) com pré-casts de tipos.
3. `etl_bq()` -> consulta `silver_cleaned_us.v_planilha_*` para DataFrame.
4. `run_etl()` -> `pd.merge(..., on='NUM_PEDIDO' or 'NUM_SOLIC',
how='inner')` e grava em `gold_analytics_us.tbl_entrada_*` com
`WRITE_TRUNCATE`.
> Observação: o DW em BigQuery funciona como réplica da planilha online de
"Controle Orçamentário Diário" (bronze+silver+gold), e o join no Python traz
atributos SQL Server para formar o conjunto final.
## 4. Data Transformation (M) no Power BI
As transformações M estão em `Programação de
Pagamento.SemanticModel/definition/tables/*.tmdl`.
### fDiario
- Fonte: BigQuery `tbl_pagamentos_consolidados`.
- Limpeza:
- `Table.Distinct(..., {"NUM_PEDIDO"})`
- `Text.Proper` em `STATUS_PEDIDO` e `UTILIZACAO`
- `Table.Sort(..., {"DATA", Order.Descending})`
### fEmergencial
- Fonte: BigQuery `tbl_entrada_emergencial`.
- Limpeza:
- `Text.Proper` em `UTILIZACAO`
- `Table.ReplaceValue(...,"Agua","Água")`
### fBacklog_Pedido
- Fonte: BigQuery `tbl_entrada_backolog`.
- Limpeza:
- `Text.Proper` em `STATUS`
- `Table.ReplaceValue(...,"GRAFICO","GRÁFICO")` e `..."QUIMICO","QUÍMICO"`
em `UTILIZACAO`
### fBacklog_Solic
- Fonte: BigQuery `tbl_entrada_backlog_solic`.
- Limpeza:
- `Table.TransformColumnTypes` (VALOR para `Currency.Type`)
- `Table.Distinct(...,{"NUM_SOLIC"})`
- mesmo tratamento de acentos/utilização do `fBacklog_Pedido`
### tbl_orcamento
- Fonte: Excel local `tbl_orcamento.xlsx`, aba `Plan2`.
- Limpeza:
- `PromoteHeaders`, `Table.TransformColumnTypes`
- substituições de valores (`Uniformes e Equip. Proteção Individual` etc)
### tabelas dimensionais
- `dUnidade`, `dFornecedor`, `dMaterial`, `dDespesa`, `dEmpresa`,
`dPreco_Material`, etc. (sem transformação agressiva, carga de base via
BigQuery e/ou datasheet)
## 5. Semantic Model (tabelas + medidas)
### Entidades principais e relações
- Fatos: `fDiario`, `fEmergencial`, `fBacklog_Pedido`, `fBacklog_Solic`,
`fEntrada_Nf`, `fSolicitacao`, `fBacklog_Solic`, `tbl_entrada_backlog_solic
(2)`.
- Dimensões: `dUnidade`, `dFornecedor`, `dMaterial`, `dDespesa`, `dEmpresa`,
`dPreco_Material`.
- Tabelas de data: `LocalDateTable_*` com hierarquias de data atribuídas a
`DATA`/`DATAPEDIDO`/`DATASOLIC`.
### Medidas DAX (tabela `Medidas`)
- `saldo`:
- Fórmula: `SUM(tbl_orcamento[ORÇAMENTO MENSAL (R$)]) -
SUM(fDiario[VALOR_TOTAL])`
- Objetivo: indicador de saldo orçamentário atual vs realizado diário.
- `saldo3`:
- Fórmula: `SUM(tbl_orcamento[ORÇAMENTO MENSAL (R$)]) -
SUM(tbl_pago_fev[VALOR])`
- Objetivo: saldo projetado descontando pagamentos de fevereiro.
- `saldo2`:
- Fórmula: `SUM(tbl_orcamento[ORÇAMENTO MENSAL (R$)]) -
SUM(tbl_pago_jan[VALOR])`
- Objetivo: saldo projetado descontando pagamentos de janeiro.
### Medidas de apoio (Design / UX)
- `Design[Frota]` e `Design[Estoque]`: strings HTML/CSS usadas como cartões
estilizados no relatório.
- `last_refresh[Last Refresh Text]`: `"Última Atualização: " &
FORMAT(MAX('last_refresh'[last_refresh]), ... )` (indicador de recência da
atualização)
## 6. Recommendations / Maintenance
### 6.1 Como adicionar um novo script
1. Copiar um dos `app_*.py` e renomear (`app_novo.py`).
2. Ajustar:
- `update_silver()` -> `v_planilha_novo`, `src_planilha_novo`.
- `etl_sql()` -> fonte SQL/joins e colunas mappings.
- `run_etl()` -> `gold_analytics_us.tbl_entrada_novo`.
3. Publicar no cron/task scheduler (`exe.bat`, pipeline do servidor).
4. Testar localmente com conexão e credenciais.
### 6.2 Alterar colunas ou lógica de fonte
- PyETL:
- Modificar `SELECT` dos `SQL_QUERY` para incluir/excluir/renomear
colunas.
- Atualizar `SAFE_CAST` em `update_silver()` para normalizar tipos.
- Power BI Model (M):
- Preferencial: abrir `Programação de Pagamento.pbip` em Power BI Desktop
e editar query em `Transform Data`.
- Manual: editar `*.tmdl` (cuidado com sintaxe M, recomenda não fazer
manualmente sem backup).
### 6.3 Credenciais e ambientes
- `projeto-suprimentos-484211-14dcca481b3a.json` no root: chave de serviço
BigQuery.
- `os.environ["GOOGLE_APPLICATION_CREDENTIALS"]` deve apontar para o JSON.
- `.env` deve conter `SQL_CONNECTION_STRING`.
### 6.4 Auditoria
- Verificar `log_execucao.txt` após cada execução de `exe.bat`.
- Conferir tabelas do BigQuery:
- `bronze_staging_us.*`
- `silver_cleaned_us.*`
- `gold_analytics_us.*`
## 7. Ponto de atenção (observações técnicas)
- `fDiario`, `fEmergencial`, `fBacklog_Pedido` e `fBacklog_Solic` usam
`GoogleBigQuery.Database(...)` no modelo, ou seja, são dependentes do acesso
à internet e permissões de projeto.
- O ativo central de integração é `tbl_pagamentos_consolidados` no DW, que
provém da planilha online de Controle Orçamentário Diário.
- A aplicação do `Text.Proper` e substituições em M garante padronização
semântica na análise.
- `saldo2`, `saldo3` podem falhar se as tabelas `tbl_pago_jan` ou
`tbl_pago_fev` não estiverem presentes no dataset; verificar se estas
tabelas residem no modelo ou se são etapas pendentes de load.
---