# Documentação Técnica - Relatório Indústria
## Visão Geral
Este documento descreve o projeto Power BI "Relatório Indústria" da Saritur,
que monitora operações industriais incluindo controle de componentes,
produtividade, carros parados e garantias.
## Data Sources
Os dados são originados das seguintes fontes:
- **Google Sheets (Planilhas Google)**:
- Controle de Componentes: 11 planilhas distintas para diferentes unidades
(Duval de Barros, Lagoa Santa, Ribeirão das Neves, Rodoviário, Transnorte,
Varginha, Lavras, Nova Lima, São Marcos, Vale do Aço, Vespasiano)
- Controle de Produção: Uma planilha para dados de produtividade
industrial
- Garantia - Indústria: Planilha para dados de garantia externa
- Garantia Carros Novos - Indústria: Planilha para dados de garantia de
carros novos
- **Arquivos Excel**:
- dDivisão.xlsx: Tabela de dimensões para divisões
- dUnidade.xlsx: Tabela de dimensões para unidades
Todas as fontes estão localizadas em diretórios acessíveis ao usuário
(Google Drive para Sheets, pasta Documents para Excel).
## Data Transformation (M)
As principais etapas de transformação nos queries Power Query incluem:
### Controle de Componentes
- **Conexão**: Cada unidade conecta a uma planilha Google Sheets específica
- **Limpeza**:
- Filtragem de linhas irrelevantes (ex: títulos)
- Promoção de cabeçalhos
- Alteração de tipos de dados (datas, números, texto)
- Filtragem de valores nulos
- **Padronização**:
- Substituição de valores vazios por nomes de localidades
- Remoção de colunas desnecessárias (MOTIVO TROCA, Nº SOLICITAÇÃO, etc.)
- Renomeação de colunas (ex: "KM ATUAL")
- **Unificação**: Todas as consultas de unidades são combinadas via
Table.Combine em fControle_Componente
### Outras Tabelas
- **fControle_Producao**: Conexão direta à planilha Google Sheets com
limpeza similar
- **Garantia**: Conexões às planilhas com transformação mínima focada em
tipos de dados
- **Dimensões**: Importação direta dos arquivos Excel com transformação
básica
## Semantic Model
O modelo contém as seguintes medidas DAX principais:
### Medidas Componentes
- **qtd total**:
`SUMX('fControle_Componente','fControle_Componente'[QUANTIDADE])` - Soma
total de quantidades solicitadas
- **media dias total**:
`AVERAGEX('fControle_Componente','fControle_Componente'[DIAS])` - Média de
dias para processamento de solicitações
- **valor total**: `SUM('fControle_Componente'[POSIÇÃO])` - Valor monetário
total das posições
### Medidas Industria
- **valor interno**: `SUMX('fControle_Producao','fControle_Producao'[VALOR
TOTAL])` - Custo interno de produção
- **valor externo**: `SUMX('fControle_Producao','fControle_Producao'[VALOR
EXTERNO])` - Custo externo de produção
- **qtd**: `SUM('fControle_Producao'[QUANTIDADE PRODUZIDA])` - Quantidade
total produzida
- **diferença**: `[valor externo] - [valor interno]` - Diferença entre
custos interno vs externo (Make-or-Buy)
- **valor_perdido_carros_novos**: `CALCULATE(SUM('GARANTIA CARROS NOVOS -
INDÚSTRIA'[VALOR]),'GARANTIA CARROS NOVOS - INDÚSTRIA'[SITUAÇÃO] =
"Improcedente")` - Valor perdido em garantias de carros novos improcedentes
- **valor_perdido_interno**: `CALCULATE(SUM('GARANTIA - INDÚSTRIA'[Valor
Perdido]),'GARANTIA - INDÚSTRIA'[Tipo_Análise] = "Interna", 'GARANTIA -
INDÚSTRIA'[Agente Causador] = "Indústria")` - Valor perdido em garantias
internas
- **valor_recuperado_externo**: `CALCULATE(SUM('GARANTIA - INDÚSTRIA'[Valor
Perdido]), 'GARANTIA - INDÚSTRIA'[Classificação] = "Procedente", 'GARANTIA -
INDÚSTRIA'[Tipo_Análise] = "Externa")` - Valor recuperado em garantias
externas procedentes
- **lucro/prejuizo**: `[valor_recuperado_externo] + [valor_perdido_interno]`
- Lucro/prejuízo geral
- **valor_perdido_total**: `CALCULATE(SUM('GARANTIA - INDÚSTRIA'[Valor
Perdido]),'GARANTIA - INDÚSTRIA'[Classificação] = "Improcedente") *-1` -
Valor total perdido (negativo)
- **valor_recuperado_total**: `CALCULATE(SUM('GARANTIA - INDÚSTRIA'[Valor
Perdido]),'GARANTIA - INDÚSTRIA'[Classificação] = "Procedente")` - Valor
total recuperado
- **lucro/prejuizo_total**: `[valor_perdido_total] +
[valor_recuperado_total]` - Lucro/prejuízo total
- **valor_recuperado_carros_novos**: `CALCULATE(SUM('GARANTIA CARROS NOVOS -
INDÚSTRIA'[VALOR]),'GARANTIA CARROS NOVOS - INDÚSTRIA'[SITUAÇÃO] =
"Procedente")` - Valor recuperado em garantias de carros novos
- **lucro/prejuizo_carrosnovos**: `[valor_recuperado_carros_novos] +
[valor_perdido_carros_novos]` - Lucro/prejuízo em carros novos
## Visual Hierarchy
O relatório contém 10 páginas, sendo 5 principais visíveis:
### Controle de Componentes (Solicitações por unidade)
Entrega ao gestor da Saritur visibilidade sobre solicitações de componentes
por unidade, incluindo:
- Quantidade total de solicitações
- Tempo médio de processamento (dias)
- Valor total envolvido
- Distribuição por tipo de solicitação, veículo, divisão e localidade
- Filtros por data e unidade
### Carros Parados (Filtro de disponibilidade)
Mostra veículos indisponíveis devido a problemas de componentes, com foco
em:
- Status de disponibilidade de veículos
- Tempo de parada
- Quilometragem atual
- Motivos de solicitação
- Análise por unidade e tipo de veículo
### Produtividade Indústria
Analisa a eficiência da produção interna versus externa, apresentando:
- Custos interno vs externo por item
- Diferença econômica (economia/poupança)
- Quantidade produzida
- Comparativos de produtividade
### Garantia Externos
Monitora garantias de veículos externos à indústria, incluindo:
- Valor perdido vs recuperado
- Classificação (Procedente/Improcedente)
- Tipo de análise (Interna/Externa)
- Agente causador
- Lucro/prejuízo total
### Garantia Novos
Focado em garantias de carros novos, com métricas específicas:
- Situação (Procedente/Improcedente)
- Valor envolvido
- Recuperação vs perda
- Análise específica para frota nova
## Maintenance
### Adicionando uma Nova Unidade
Para adicionar uma nova unidade ao sistema de Controle de Componentes:
1. **Criar nova planilha Google Sheets**:
- Copie o formato de uma unidade existente
- Mantenha a estrutura de colunas consistente
- Compartilhe a planilha com as credenciais do Power BI
2. **Atualizar Power Query**:
- No Power BI Desktop, vá para "Editar Consultas"
- Duplique uma consulta existente de unidade
- Altere a URL da fonte para a nova planilha
- Ajuste filtros e substituições conforme necessário (ex: nome da
localidade)
- Renomeie a consulta para "CONTROLE COMPONENTE - [NOVA UNIDADE]"
3. **Atualizar fControle_Componente**:
- Edite a consulta fControle_Componente
- Adicione a nova consulta à lista no Table.Combine
- Certifique-se de que os tipos de dados sejam compatíveis
4. **Atualizar dUnidade.xlsx**:
- Adicione a nova unidade ao arquivo Excel de dimensões
- Inclua código da unidade e nome
5. **Testar e Publicar**:
- Atualize o modelo
- Verifique se os filtros e relacionamentos funcionam
- Publique no Power BI Service