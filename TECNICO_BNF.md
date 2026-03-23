# Documentação Técnica - Projeto BNF (Power BI)
## Visão Geral
Este projeto Power BI (.pbip) analisa dados relacionados ao BNF (Biodiesel
Nacional Fiscal) para a empresa Saritur e suas subsidiárias. O dashboard
fornece insights sobre valores mensais e totais de BNF, entradas de óleo
biodiesel, e percentuais de cumprimento de metas.
## Data Sources
Os dados são originados de duas fontes principais:
1. **Excel File**:
 - Arquivo: `C:\Users\Anl-suprimentos\Documents\BNF.xlsx`
 - Sheet: "BNF"
 - Tabela: `fBNF`
 - Contém dados de empresas, CNPJ, valores totais e mensais de BNF, e
datas de início e fim.
2. **SQL Server Database**:
 - Servidor: `192.168.0.11`
 - Banco: `BASE_STAGE_OFICINASMATERIAIS`
 - Tabelas extraídas via queries SQL:
 - `fEntrada`: Dados de entradas de notas fiscais para itens específicos
de óleo biodiesel (S10 e S500), filtrados por data > '2025-11-01'.
 - `dEmpresa`: Dimensão de empresas, com JOIN entre `DIM_CadLocal` e
`DIM_EMPRESA_FILIAL`, excluindo certas empresas.
 - `dUnidade`: (Similar, assumindo estrutura parecida para unidades).
## Data Transformation
### Em M (Power Query):
- **fBNF**:
 - Carregar dados do Excel.
 - Promover cabeçalhos.
 - Alterar tipos de colunas (text, Int64, date).
 - Transformar coluna EMPRESA para "Proper Case" (cada palavra em
maiúscula).
- **fEntrada**:
 - Executar query SQL para selecionar e converter tipos.
 - Alterar tipo da coluna ENTRADANF para date.
- **dEmpresa**:
 - Executar query SQL com JOIN e filtros.
 - Transformar NOME_EMPRESA para "Proper Case".
 - Remover duplicatas baseadas em COD_EMPRESA.
- **dUnidade**:
 - (Assumindo similar a dEmpresa, com query SQL para unidades).
### Em SQL:
- Queries diretas no banco para filtrar dados relevantes:
 - Filtrar entradas por data > '2025-11-01' e itens específicos de óleo
biodiesel.
 - JOINs para combinar dados de localização e empresa.
 - Exclusões de empresas específicas.
Não há Appends ou Joins explícitos em M além dos SQL JOINs; as
transformações focam em limpeza de tipos e texto.
## Semantic Model
### Medidas DAX Principais:
1. **diferenca**:
 - Fórmula: `SUM('fBNF'[VALOR_MENSAL]) - SUM('fEntrada'[QTDE_ITEM])`
 - Lógica: Calcula a diferença entre o valor mensal de BNF e a quantidade
de itens entrados.
 - Objetivo de negócio: Identificar o gap entre o planejado (BNF mensal) e
o realizado (entradas), ajudando a monitorar desvios em metas de biodiesel.
2. **porcentagem_mensal**:
 - Fórmula: `DIVIDE(SUM('fEntrada'[QTDE_ITEM]),
SUM('fBNF'[VALOR_MENSAL]))`
 - Lógica: Divide a soma das quantidades entradas pela soma dos valores
mensais de BNF.
 - Objetivo de negócio: Medir o percentual de cumprimento mensal das metas
de BNF, indicando eficiência operacional.
3. **porcentagem_total**:
 - Fórmula: `DIVIDE(SUM('fEntrada'[QTDE_ITEM]), SUM('fBNF'[VALOR_TOTAL]))`
 - Lógica: Divide a soma das quantidades entradas pela soma dos valores
totais de BNF.
 - Objetivo de negócio: Avaliar o progresso geral em relação ao total
planejado, fornecendo uma visão acumulada do desempenho.
## Visual Hierarchy
O dashboard é estruturado em páginas principais e tooltips para
detalhamento:
1. **BNF Mensal**:
 - Entrega ao gestor: Visão mensal dos valores de BNF e entradas, com foco
em performance corrente. Permite monitorar se as metas mensais estão sendo
atingidas, usando medidas como porcentagem_mensal e diferenca.
2. **BNF Total**:
 - Entrega ao gestor: Visão acumulada dos totais de BNF e entradas. Ajuda
a avaliar o progresso geral do ano ou período, com ênfase em
porcentagem_total para compliance total.
3. **Tooltips - Saritur Mensal/Total**:
 - Detalhes específicos para a empresa Saritur, mostrando dados mensais ou
totais em pop-ups para visuais principais.
4. **Tooltips - Turilessa Mensal/Total**:
 - Detalhes específicos para a empresa Turilessa, mostrando dados mensais
ou totais em pop-ups para visuais principais.
5. **Tooltips - S&M Mensal/Total**:
 - Detalhes específicos para a empresa S&M, mostrando dados mensais ou
totais em pop-ups para visuais principais.
6. **Tooltips - Coletivos Mensal/Total**:
 - Detalhes específicos para a empresa Coletivos, mostrando dados mensais
ou totais em pop-ups para visuais principais.
7. **Tooltips - Companhia**:
 - Detalhes específicos para a empresa Companhia, mostrando dados mensais
ou totais em pop-ups para visuais principais.
As páginas de tooltip permitem drill-down sem sobrecarregar as páginas
principais, facilitando a navegação para gestores focados em subsidiárias
específicas.
## Maintenance
### Reinicialização de Contagem ao Chegar o Prazo
- **Cenário**: Quando a DATA_FINAL de um contrato de BNF for atingida, a
contagem precisa ser reiniciada para o próximo período.
- **Instruções para o sucessor**:
 1. Verificar no modelo se há registros com DATA_FINAL expirada.
 2. Atualizar o arquivo Excel `BNF.xlsx` com novos valores e datas para o
próximo ciclo.
 3. No SQL Server, ajustar filtros de data (atualmente > '2025-11-01') para
o novo período.
 4. Recarregar o modelo semântico no Power BI Desktop.
 5. Publicar e atualizar o dataset no Power BI Service.
 6. Testar medidas DAX para garantir que diferenças e percentuais reflitam
o novo ciclo.
 7. Notificar gestores sobre a reinicialização e treinar no uso das novas
métricas.