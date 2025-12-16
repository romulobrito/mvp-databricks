# Problema e Metodologia - MVP Análise de Conformidade de Ciclos

**Nota**: Este documento descreve o problema de negócio, metodologia de análise e base teórica do MVP. Para detalhes técnicos completos sobre transformações, justificativas e implementação do pipeline ETL, consulte [PIPELINE_ETL.md](PIPELINE_ETL.md).

## 1. Contextualização do Problema

### 1.1 Contexto Institucional

A Agência Nacional de Transportes Terrestres (ANTT) é responsável pela regulação e fiscalização do transporte rodoviário de passageiros e cargas no Brasil. Uma das atribuições da ANTT é monitorar a operação de serviços de patrulhamento rodoviário em rodovias federais, garantindo que as concessionárias cumpram os requisitos contratuais estabelecidos.

### 1.2 Problema de Negócio

As concessionárias de rodovias federais devem manter viaturas de patrulhamento operando em trechos específicos das rodovias, garantindo frequência mínima de passagem para atender a segurança dos usuários. O contrato estabelece que cada trecho (identificado por quilometragem - KM) deve ser percorrido por pelo menos uma viatura em intervalos regulares ao longo das 24 horas do dia.

O problema central é: **Como verificar se as viaturas estão cumprindo os requisitos contratuais de frequência de passagem?**

### 1.3 Desafios Técnicos

1. **Volume de Dados**: Milhões de registros de GPS gerados diariamente pelas viaturas
2. **Complexidade Temporal**: Necessidade de analisar padrões ao longo de múltiplos ciclos de 3 horas
3. **Cobertura Espacial**: Análise de centenas de trechos (KMs) diferentes
4. **Multiplicidade de Viaturas**: Diferentes viaturas operando em horários e trechos distintos
5. **Regras de Conformidade**: Limites individuais por ciclo e limites gerais acumulados por período

### 1.4 Impacto Esperado

A solução deste problema permite:
- **Fiscalização Eficiente**: Verificação automática do cumprimento contratual
- **Tomada de Decisão**: Identificação de trechos e períodos com maior risco de não conformidade
- **Otimização de Recursos**: Análise de desempenho por viatura para melhor distribuição
- **Transparência**: Evidências objetivas para avaliação de desempenho das concessionárias

## 2. Definição do Problema

### 2.1 Problema Principal

**Verificar a conformidade de frequência de passagem de viaturas de patrulhamento em trechos específicos de rodovias federais, organizados em ciclos de 3 horas ao longo das 24 horas do dia.**

### 2.2 Objetivos Específicos

1. Calcular a taxa de conformidade por ciclo de 3 horas
2. Avaliar a conformidade por período (P1: madrugada/manhã, P2: tarde/noite)
3. Identificar trechos com maior risco de não conformidade
4. Analisar distribuição temporal da conformidade
5. Avaliar regularidade operacional por viatura

### 2.3 Perguntas de Negócio

#### Pergunta 1: Qual a taxa de conformidade por ciclo?

**Justificativa**: Cada ciclo de 3 horas deve ter pelo menos uma passagem de viatura. É necessário verificar se a média de posição dentro do ciclo está dentro do limite estabelecido (180 minutos).

**Método de Resposta**: Calcular a média mensal da posição no ciclo (`Minuto_no_Ciclo`) para cada combinação de `Mês x KM x Ciclo`, comparar com o limite de 180 minutos e calcular a taxa de conformidade.

#### Pergunta 2: Qual a taxa de conformidade por período (P1/P2)?

**Justificativa**: Além da conformidade individual por ciclo, existe um limite geral acumulado por período (soma dos 4 ciclos). É necessário verificar se o saldo acumulado está dentro do limite de 60 minutos.

**Método de Resposta**: Somar os saldos dos 4 ciclos de cada período, comparar com o limite de 60 minutos e calcular a taxa de conformidade por período.

#### Pergunta 3: Quais trechos (KMs) apresentam maior risco de não conformidade?

**Justificativa**: Identificar trechos críticos permite priorizar ações de fiscalização e correção.

**Método de Resposta**: Calcular o saldo médio por KM e identificar trechos com maior saldo (mais próximos do limite ou acima dele).

#### Pergunta 4: Qual a distribuição temporal da conformidade?

**Justificativa**: Entender padrões temporais ajuda a identificar períodos de maior ou menor eficiência operacional.

**Método de Resposta**: Comparar taxas de conformidade e médias de posição entre diferentes ciclos e períodos.

#### Pergunta 5: Quais viaturas apresentam maior regularidade?

**Justificativa**: Identificar viaturas com melhor desempenho permite replicar boas práticas e otimizar distribuição de recursos.

**Método de Resposta**: Calcular desvio padrão da posição no ciclo por viatura e identificar as mais regulares.

## 3. Metodologia de Análise

### 3.1 Arquitetura de Dados

O MVP utiliza uma arquitetura de **Data Lake** com três camadas:

```
Bronze (Raw) -> Silver (Cleaned/Transformed) -> Gold (Aggregated/Analytical)
```

**Justificativa da Escolha**:
- **Data Lake** é mais adequado para dados não estruturados e alta variabilidade
- Permite armazenar dados brutos sem schema rígido inicialmente
- Facilita evolução do modelo conforme necessidades mudam
- Suporta análises exploratórias e processamento em lote

### 3.2 Racional de Conformidade

#### 3.2.1 Conceito de Ciclos

O dia é dividido em **8 ciclos de 3 horas**:

| Ciclo | Horário | Período |
|-------|---------|---------|
| 1 | 00:00 - 02:59 | P1 |
| 2 | 03:00 - 05:59 | P1 |
| 3 | 06:00 - 08:59 | P1 |
| 4 | 09:00 - 11:59 | P1 |
| 5 | 12:00 - 14:59 | P2 |
| 6 | 15:00 - 17:59 | P2 |
| 7 | 18:00 - 20:59 | P2 |
| 8 | 21:00 - 23:59 | P2 |

**Períodos**:
- **P1**: Ciclos 1-4 (madrugada/manhã: 00:00 - 11:59)
- **P2**: Ciclos 5-8 (tarde/noite: 12:00 - 23:59)

#### 3.2.2 Métrica de Posição no Ciclo

Para cada registro de viatura, calcula-se a **posição dentro do ciclo**:

```
Minuto_no_Ciclo = Minutos_desde_00h - Inicio_Ideal_do_Ciclo
```

Onde:
- `Minutos_desde_00h` = hora * 60 + minuto + segundo/60
- `Inicio_Ideal_do_Ciclo` = (Ciclo - 1) * 180

**Exemplo**: Uma viatura que passa as 05:30 (330 minutos desde 00h) no Ciclo 2:
- `Inicio_Ideal` = (2-1) * 180 = 180 minutos
- `Minuto_no_Ciclo` = 330 - 180 = 150 minutos

#### 3.2.3 Seleção de Registros de Referência

Como uma viatura pode passar múltiplas vezes no mesmo ciclo, é necessário selecionar **1 registro representativo por combinação de chaves**.

**Chaves de Agrupamento**: `Data x Ciclo x Rodovia x Sentido x KM`

**Lógica de Seleção**:
1. **Preferência**: Priorizar registros que passaram **antes ou exatamente no fim ideal do ciclo** (`Minutos <= Fim_Ideal`)
2. **Proximidade**: Entre registros da mesma preferência, escolher o mais próximo do `Fim_Ideal`
3. **Fallback**: Se não houver registro antes do fim ideal, escolher o mais próximo após o fim ideal
4. **Desempate**: Em caso de empate, escolher o registro com menor timestamp (`Marcacao`)

**Justificativa**: Esta lógica garante que estamos selecionando o registro mais representativo de cada ciclo, priorizando registros que demonstram cumprimento adequado do ciclo. A seleção é feita independente da viatura, garantindo que cada dia/ciclo/km tenha exatamente 1 registro de referência para cálculo da média mensal.

#### 3.2.4 Limites de Conformidade

**Limite Individual (por Ciclo)**:
- Cada ciclo deve ter `Media_Posicao_no_Ciclo <= 180` minutos
- Calculado como média mensal da `Minuto_no_Ciclo` para cada `Mês x KM x Ciclo`

**Limite Geral (por Período)**:
- Cada período deve ter `Saldo_Periodo <= 60` minutos
- `Saldo_Periodo` = soma dos 4 `Saldo_Ciclo` do período
- `Saldo_Ciclo` = `Media_Posicao_no_Ciclo - 180`

**Interpretação dos Saldos**:
- **Saldo Negativo**: Indica margem de segurança (operação abaixo do limite)
- **Saldo Positivo**: Indica excedente (operação acima do limite)
- **Saldo Zero**: Operação exatamente no limite

### 3.3 Pipeline de Processamento

**Nota**: Para detalhes técnicos completos de cada etapa, consulte [PIPELINE_ETL.md](PIPELINE_ETL.md).

#### Etapa 0: Bronze - Concatenação de Dados Brutos

**Notebook**: `00_gerar_bronze_ciclos_concatenados.ipynb`

**Entrada**: Arquivos Excel (`Ciclo01.xlsx`, `Ciclo02.xlsx`)

**Processamento**:
- Leitura dos arquivos Excel usando `pandas.read_excel()`
- Concatenação vertical com `pd.concat(ignore_index=True)`
- Preservação de todas as colunas originais
- Conversão para formato Parquet para eficiência

**Saída**: `dados_bronze/ciclos_concatenados.parquet`

**Volume**: ~1.548.566 registros

**Justificativa**: Manter dados brutos permite reprocessamento futuro e auditoria. O formato Parquet oferece compressao e leitura eficiente.

#### Etapa 1: Análise de Qualidade de Dados

**Notebook**: `01_analise_qualidade_dados.ipynb`

**Entrada**: Dados de qualquer camada (Bronze, Silver Base, Silver Referencia)

**Processamento**:
- Análise de completude (valores nulos)
- Validação de consistência (domínios esperados)
- Verificação de integridade (duplicatas)
- Análise de precisão (valores inválidos, outliers)
- Validação de impacto nas perguntas de negócio

**Saída**: Relatório textual de qualidade (impresso no notebook)

**Justificativa**: Identificar problemas nos dados antes de prosseguir com análises. Pode ser executado após cada camada para validação incremental.

#### Etapa 2: Silver Base - Tratamento Inicial

**Notebook**: `02_gerar_silver_ciclos_base.ipynb`

**Entrada**: `dados_bronze/ciclos_concatenados.parquet`

**Transformações**:
1. Renomeação de colunas para padronização
2. Criação de timestamp (`Marcacao`) combinando data e hora
3. Conversão de KM de formato string ("XXX+YYY") para float (XXX.YYY)
4. Cálculo de colunas derivadas:
   - `Data`: Data extraída do timestamp
   - `Ciclo`: Número do ciclo (1-8) calculado como `hora // 3 + 1`
   - `Minutos`: Minutos desde 00:00
   - `Inicio_Ideal`: `(Ciclo - 1) * 180`
   - `Fim_Ideal`: `Ciclo * 180`

**Saída**: `silver/silver_ciclos_base.parquet`

**Volume**: ~1.548.566 registros (mantido)

**Justificativa**: Criar colunas necessárias para análise de conformidade mantendo todos os registros originais.

#### Etapa 3: Silver Referência - Seleção de Registros

**Notebook**: `03_gerar_silver_ciclos_referencia.ipynb`

**Entrada**: `silver/silver_ciclos_base.parquet`

**Transformações**:
1. **Filtros Prévios**:
   - Filtro por rodovia (ex: "BR-153/GO")
   - Filtro por sentido (ex: "S")
   - Filtro de tolerância de tempo (±15 minutos em relação ao ciclo)

2. **Lógica de Seleção**:
   - Cálculo de `Preferencia` (0 = antes/igual ao fim ideal, 1 = depois)
   - Cálculo de `Delta_limite` (distância até o fim ideal)
   - Ordenação: `Data, Ciclo, Rodovia, Sentido, KM, Preferencia, Delta_limite, Marcacao`
   - Seleção: `.groupby(Data, Ciclo, Rodovia, Sentido, KM).first()` (primeiro registro de cada grupo)

3. **Cálculo de Posição no Ciclo**:
   - `Minuto_no_Ciclo` = `Minutos - Inicio_Ideal`

**Saída**: `silver/silver_ciclos_referencia.parquet`

**Volume**: ~197.417 registros (reducao de ~87%)

**Justificativa**: Reduzir volume de dados selecionando apenas registros representativos (1 por Data x Ciclo x KM), mantendo informação suficiente para avaliação de conformidade mensal.

#### Etapa 4: Gold - Agregação de Conformidade

**Notebook**: `04_gerar_gold_conformidade_racional.ipynb`

**Entrada**: `silver/silver_ciclos_referencia.parquet`

**Transformações**:

1. **Agregação por Ciclo**:
   - Agrupa por `Mês x KM x Ciclo x Período`
   - Calcula `Media_Posicao_no_Ciclo` = média mensal de `Minuto_no_Ciclo`
   - Calcula `Saldo_Ciclo` = `Media_Posicao_no_Ciclo - 180`
   - Define `Status_Ciclo` = "CONFORME" se `Media_Posicao_no_Ciclo <= 180`, senão "NAO CONFORME"

2. **Agregação por Período**:
   - Agrupa por `Mês x KM x Período`
   - Calcula `Saldo_Periodo` = soma dos 4 `Saldo_Ciclo` do período
   - Define `Status_Periodo` = "CONFORME" se `Saldo_Periodo <= 60`, senão "NAO CONFORME"

**Saída**: 
- `gold/gold_conformidade_ciclo_km.parquet` (~140.960 linhas)
- `gold/gold_conformidade_periodo_km.parquet` (~119.117 linhas)
- Versões Excel para visualização

**Justificativa**: Criar tabelas agregadas prontas para análise, reduzindo complexidade de consultas futuras. As versões Excel facilitam visualização e compartilhamento.

#### Etapa 5: Painel Consolidado Multi-KM

**Notebook**: `05_gerar_painel_consolidado_mvp.ipynb`

**Entrada**: 
- `gold/gold_conformidade_ciclo_km.parquet`
- `gold/gold_conformidade_periodo_km.parquet`
- `dados_bronze/ciclos_concatenados.parquet` (para selecao de top KMs)

**Processamento**:
1. Seleção automática dos top N KMs por frequência de registros (lógica similar a `listar_top_kms.py`)
2. Filtro dos dados Gold para os KMs selecionados
3. Montagem de matrizes por período (P1: ciclos 1-4, P2: ciclos 5-8)
4. Cálculo de médias entre KMs por ciclo
5. Cálculo de atendimento e saldos por ciclo e período
6. Geração de resumo "Cenário Concessionária"

**Saída**: `gold/painel_consolidado_mvp_YYYYMM.xlsx`

**Formato**: Excel com abas P1 e P2, cada uma contendo:
- Linhas: Ciclos (4 linhas por período)
- Colunas: KMs alvo + Média + Atendimento + Saldo
- Linha de resumo: Cenário Concessionária

**Justificativa**: Criar visão executiva consolidada multi-KM no formato próximo ao painel original, facilitando interpretação e tomada de decisão.

**Pré-processamento de KMs Alvo**:
- Seleção automática dos top N KMs por frequência de registros na rodovia e sentido específicos
- Garante representatividade estatística adequada
- Metodologia flexível: qualquer lista de KMs pode ser utilizada
- Lógica similar ao script `listar_top_kms.py` para identificação dos trechos mais significativos

#### Etapa 6: Análise da Solução do Problema

**Notebook**: `06_analise_solucao_problema.ipynb`

**Entrada**: 
- `gold/gold_conformidade_ciclo_km.parquet`
- `gold/gold_conformidade_periodo_km.parquet`
- `silver/silver_ciclos_referencia.parquet`

**Processamento**:
1. Resposta às 5 perguntas de negócio definidas no objetivo
2. Cálculo de taxas de conformidade por ciclo e período
3. Identificação de trechos com maior risco
4. Análise de distribuição temporal
5. Avaliação de regularidade por viatura
6. Geração de visualizações (gráficos PNG)
7. Discussão técnica e interpretação dos resultados

**Saída**: 
- Relatório textual com respostas e discussão
- Gráficos salvos em `visualizacoes/`:
  - `pergunta1_conformidade_ciclo.png`
  - `pergunta2_conformidade_periodo.png`
  - `pergunta3_trechos_risco.png`
  - `pergunta4_distribuicao_temporal.png`
  - `pergunta5_regularidade_viaturas.png`

**Justificativa**: Responder objetivamente às perguntas de negócio com evidências quantitativas e visualizações claras, fornecendo base sólida para tomada de decisões gerenciais.

### 3.4 Métodos de Análise

#### 3.4.1 Análise de Qualidade de Dados

**Objetivo**: Identificar problemas nos dados que possam afetar as respostas das perguntas de negócio.

**Métodos Aplicados**:
- **Completude**: Verificação de valores nulos por atributo em cada camada
- **Consistência**: Validação de valores dentro dos domínios esperados (ciclos [1-8], minutos [0-1440), etc.)
- **Integridade**: Verificação de duplicatas completas na camada Bronze
- **Precisão**: Validação de valores inválidos e outliers
- **Impacto nas Perguntas**: Análise de quais atributos são críticos para cada pergunta de negócio

**Ferramentas**: Pandas para análise estatística descritiva e validações.

**Execução**: O notebook de análise de qualidade pode ser executado após cada camada (Bronze, Silver Base, Silver Referência) para validação incremental da qualidade dos dados.

#### 3.4.2 Análise de Conformidade

**Objetivo**: Responder às perguntas de negócio definidas no objetivo.

**Métodos Aplicados**:
- **Agregações**: `groupby` para calcular médias e somas por dimensões
- **Comparações**: Comparação de valores com limites contratuais
- **Rankings**: Identificação de top N trechos/viaturas por critérios específicos
- **Distribuições**: Análise de distribuições temporais e espaciais

**Ferramentas**: Pandas para manipulação de dados, NumPy para cálculos numéricos.

#### 3.4.3 Visualização e Apresentação

**Objetivo**: Apresentar resultados de forma clara e acionável.

**Métodos Aplicados**:
- **Painel Consolidado Multi-KM**: Planilha Excel com estrutura similar ao formato original, contendo:
  - Abas separadas por período (P1 e P2)
  - Matrizes de ciclos x KMs com médias de posição no ciclo
  - Colunas de resumo (Média entre KMs, Atendimento, Saldo)
  - Linha de resumo "Cenário Concessionária" por período
- **Tabelas Agregadas**: Tabelas Gold em formato Parquet (eficiência) e Excel (visualização)
- **Visualizações Gráficas**: Gráficos PNG gerados para cada pergunta de negócio:
  - Gráficos de barras para taxas de conformidade
  - Histogramas para distribuições
  - Gráficos comparativos entre períodos
- **Relatórios Textuais**: Saídas dos notebooks com discussão técnica e interpretação dos resultados

**Ferramentas**: 
- Pandas para geração de Excel e manipulação de dados
- Matplotlib e Seaborn para geração de gráficos
- OpenPyXL para escrita de arquivos Excel
- Formatação de texto para relatórios

### 3.5 Validação e Qualidade

#### 3.5.1 Validações de Processamento

- Verificação de existência de arquivos de entrada
- Contagem de registros antes e após cada transformação
- Validação de tipos de dados
- Verificação de valores dentro de domínios esperados

#### 3.5.2 Validações de Negócio

- Verificação de que `Minuto_no_Ciclo` está no domínio [0, 180)
- Validação de que ciclos estão no domínio [1, 8]
- Verificação de que saldos são calculados corretamente
- Validação de que status de conformidade corresponde aos limites

#### 3.5.3 Rastreabilidade

- Preservação de `KM_original` para auditoria
- Logs detalhados em cada etapa do pipeline
- Estrutura de diretórios clara (Bronze/Silver/Gold)
- Documentação de todas as transformações aplicadas

### 3.6 Limitações e Considerações

#### 3.6.1 Limitações dos Dados

- Dados disponíveis apenas para um mês (maio/2025)
- Filtros aplicados limitam análise a uma rodovia e sentido específicos
- Seleção de referência reduz volume mas pode perder informações de frequência

#### 3.6.2 Limitações da Metodologia

- Análise focada em conformidade binária (conforme/não conforme)
- Não considera fatores externos (clima, tráfego, manutenção)
- Agregação mensal pode ocultar variações diárias importantes

#### 3.6.3 Considerações para Produção

- Pipeline atual é adequado para análise exploratória e MVP
- Notebooks já adaptados para funcionar no ambiente Databricks:
  - Caminhos configurados para workspace do Databricks
  - Compatibilidade com DBFS público desabilitado
  - Formato Parquet mantido para compatibilidade com Delta Lake
- Para produção em escala, seria necessário:
  - Conversão para PySpark para escalabilidade
  - Criação de tabelas Delta Lake nas camadas Bronze, Silver e Gold
  - Automação do pipeline via Databricks Workflows (ETL agendado)
  - Processamento incremental (apenas novos dados)
  - Alertas automáticos para não conformidades
  - Dashboard interativo com Databricks SQL ou visualizações Python
  - Monitoramento de qualidade de dados automatizado

## 4. Estrutura de Dados

### 4.1 Modelo Conceitual

```
Dados Brutos (Excel)
    |
    v
Bronze: Dados Concatenados (Parquet)
    |
    |---> Analise de Qualidade (Relatorio Textual)
    |
    v
Silver Base: Dados Tratados (Parquet)
    |
    |---> Analise de Qualidade (Relatorio Textual)
    |
    v
Silver Referencia: Registros Selecionados (Parquet)
    |
    |---> Analise de Qualidade (Relatorio Textual)
    |
    v
Gold: Conformidade Agregada (Parquet + Excel)
    |
    |---> Gold Ciclo: Conformidade por Ciclo
    |---> Gold Periodo: Conformidade por Periodo
    |
    v
Painel Consolidado Multi-KM (Excel)
    |
    v
Analise da Solucao (Relatorio + Visualizacoes)
```

**Nota**: A Análise de Qualidade pode ser executada após cada camada para validação incremental.

### 4.2 Relacionamentos entre Tabelas

- **Bronze -> Silver Base**: 1:1 (todos os registros são tratados)
- **Silver Base -> Silver Referência**: N:1 (múltiplos registros -> 1 registro por chave)
- **Silver Referência -> Gold Ciclo**: N:1 (agregação por Mês x KM x Ciclo)
- **Gold Ciclo -> Gold Período**: N:1 (agregação por Mês x KM x Período)

### 4.3 Granularidades

- **Bronze**: 1 registro = 1 evento GPS de viatura
- **Silver Base**: 1 registro = 1 evento GPS tratado
- **Silver Referência**: 1 registro = 1 passagem representativa por Data x Ciclo x Rodovia x Sentido x KM
- **Gold Ciclo**: 1 registro = 1 agregação mensal por Mês x KM x Ciclo x Período
- **Gold Período**: 1 registro = 1 agregação mensal por Mês x KM x Período
- **Painel Consolidado**: 1 linha = 1 ciclo (4 linhas por período) + 1 linha de resumo por período
- **Análise da Solução**: Agregações adicionais para responder perguntas de negócio (por ciclo, período, KM, viatura)

## 5. Metodologia de Avaliação

### 5.1 Métricas de Conformidade

**Métrica Individual (Ciclo)**:
```
Conformidade_Ciclo = Media_Posicao_no_Ciclo <= 180 minutos
```

**Métrica Geral (Período)**:
```
Conformidade_Periodo = Saldo_Periodo <= 60 minutos
```

### 5.2 Cálculo de Taxas

**Taxa de Conformidade por Ciclo**:
```
Taxa = (Número de combinações Mês x KM x Ciclo conformes) / (Total de combinações) * 100
```

**Taxa de Conformidade por Período**:
```
Taxa = (Número de combinações Mês x KM x Período conformes) / (Total de combinações) * 100
```

### 5.3 Interpretação dos Resultados

- **100% Conforme**: Todos os ciclos/períodos atendem aos limites
- **< 100% Conforme**: Alguns ciclos/períodos excedem os limites
- **Saldo Negativo**: Margem de segurança (bom desempenho)
- **Saldo Positivo**: Excedente (necessita atenção)

## 6. Referências e Base Teórica

### 6.1 Conceitos de Data Lake

- **Bronze Layer**: Dados brutos sem tratamento
- **Silver Layer**: Dados limpos e enriquecidos
- **Gold Layer**: Dados agregados para análise

### 6.2 Conceitos de ETL

- **Extração**: Leitura de dados de fontes diversas
- **Transformação**: Aplicação de regras de negócio e limpeza
- **Carga**: Persistência em formato otimizado para análise

### 6.3 Conceitos de Qualidade de Dados

- **Completude**: Presença de valores em todos os campos
- **Consistência**: Valores dentro dos domínios esperados
- **Integridade**: Ausência de duplicatas e inconsistência referencial
- **Precisão**: Valores corretos e válidos

## 7. Próximos Passos Metodológicos

### 7.1 Melhorias Futuras

1. **Análise Preditiva**: Modelos para prever risco de não conformidade
2. **Análise de Causalidade**: Investigar fatores que influenciam conformidade
3. **Análise Comparativa**: Comparar desempenho entre diferentes rodovias
4. **Análise em Tempo Real**: Processamento de dados em tempo real para alertas

### 7.2 Expansão da Metodologia

1. **Multi-Rodovia**: Expandir análise para todas as rodovias simultaneamente
2. **Multi-Mês**: Análise histórica de tendências ao longo do tempo
3. **Fatores Externos**: Incorporar dados de clima, tráfego, eventos
4. **Otimização**: Modelos de otimização para distribuição de recursos

