# Documentação Técnica - Relatório Compras (Power BI)
## Visão Geral
Este documento descreve o projeto Power BI "Relatório Compras" da Saritur,
focado em análises de compras, estoque, cotações e produtividade.
## Data Sources
### Principais Fontes de Dados
- **Banco de Dados SQL Server**: Servidor `192.168.0.11`, base
`BASE_STAGE_OFICINASMATERIAIS`
- **Tabelas Principais**:
 - `STAGINGPEDIDO_COMPRASCOMPLETO` (fCompras)
 - `VWPW_LEADTIME_MAT_IMEDIATO` (lead times)
 - `FACT_CPR_SOLIC_COTACAO_PEDIDO` (fCotacao)
 - `DIM_CadMaterial` (dMaterial)
 - `DIM_CadLocal` (dUnidade)
 - `DIM_CadFornecedor` (dFornecedor)
 - `DIM_DESPESA` (dDespesa)
 - `DIM_EMPRESA` (dEmpresa)
### Tipo de Conexão
- Modo Import (dados armazenados no modelo)
- Queries M com SQL nativo embutido
## Data Transformation
### Limpeza e Unificação em M (Power Query)
- **Conversão de Tipos**: Colunas de data, moeda e numéricas convertidas
para tipos apropriados (e.g., `VALOR_TOTAL` para Currency)
- **Ordenação**: Dados ordenados por data decrescente para análises
temporais
- **Transformações de Texto**: Aplicação de `Text.Proper` para padronizar
nomes (e.g., fornecedores e materiais)
### Transformações em SQL
- **Joins**: LEFT JOIN entre tabelas de compras e lead time para calcular
tempos de entrega
- **Cálculos de Lead Time**:
 - `LEAD_TIME_COMPLETO`: DATEDIFF(day, DATA_SOLIC, DATA_ENTRADA_NF)
 - `LEAD_TIME_COMPRAS`: DATEDIFF(day, DATA_SOLIC, DATA_PEDIDO)
 - `LEAD_TIME_FORNECEDOR`: DATEDIFF(day, DATA_PEDIDO, DATA_ENTRADA_NF)
- **Subqueries para Média Acumulada**: Cálculo de média histórica de preços
por material até a data do pedido
- **Filtros**: Dados filtrados para pedidos após '2025-01-01'
## Semantic Model
### Medidas DAX Principais
#### Medidas Básicas
- `qtd_comprada`: SUM(fCompras[QUANTIDADE]) - Total de quantidade comprada
- `compra_total`: SUM(fCompras[VALOR_TOTAL]) - Valor total das compras
- `media_2025`: DIVIDE([compra_total], 12) - Média mensal de compras para
2025
#### Análises de Preço
- `valor minimo`: MIN(fCompras[VALOR_UNITARIO]) - Menor preço unitário
- `valor maximo`: MAX(fCompras[VALOR_UNITARIO]) - Maior preço unitário
- `valor total maximo`: [qtd_comprada] * [valor maximo] - Custo se comprado
ao preço máximo
- `valor total minimo`: [qtd_comprada] * [valor minimo] - Custo se comprado
ao preço mínimo
- `diferença_minimo_maximo`: [valor maximo] - [valor minimo] - Amplitude de
preços
#### Média Acumulada
- `Media Acumulada`: Calcula a média histórica de preços por material até a
data atual, considerando apenas compras anteriores
#### Análises de Economia (Foco Produtividade/Economia)
- `Menor Valor`: MINX(FILTER(ALL(fCompras), fCompras[COD_MATERIAL_INTERNO] =
SELECTEDVALUE(fCotacao[COD_MATERIAL_INT])), fCompras[VALOR_UNITARIO]) -
Menor preço praticado para o material selecionado
- `Fornecedor_Campeao`: Identifica o fornecedor que ofereceu o menor preço
para o material, via LOOKUPVALUE na dimensão dFornecedor
- `Data_Menor_Preco`: Data da compra com o menor preço para o material
### Objetivo de Negócio das Medidas de Economia
- **Identificação de Oportunidades de Economia**: As medidas permitem
comparar preços históricos e identificar fornecedores mais competitivos
- **Análise de Produtividade**: Avaliar eficiência nas negociações, medindo
a diferença entre preços máximo e mínimo praticados
- **Benchmarking**: Comparar desempenho de compras contra médias históricas
para otimizar decisões futuras
## Visual Hierarchy
### Estrutura das Páginas
1. **Página Inicial**: Dashboard principal com visão geral das compras da
Saritur
2. **2025**: Análises detalhadas de compras para o ano de 2025
3. **2026**: Análises detalhadas de compras para o ano de 2026
4. **S&M**: Previsto x Realizado
5. **ENSS**: Previsto x Realizado
6. **Histórico Compras**: Histórico temporal das compras
7. **Gestão de Cotações**: Controle e análise de processos de cotação
8. **Estoque**: Gestão de inventário e estoques
9. **Lead Time**: Análises de tempo de entrega e eficiência logística
10. **tbl**: Tabela oculta para desenvolvimento