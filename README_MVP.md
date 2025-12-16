# MVP - Pipeline de Dados: Analise de Conformidade de Ciclos de Inspecao

## Documentacao do MVP

Este MVP possui documentacao completa organizada em varios arquivos:

- **[PROBLEMA_E_METODOLOGIA.md](PROBLEMA_E_METODOLOGIA.md)**: Documento completo com contextualizacao do problema, metodologia detalhada, e base teorica
- **[CATALOGO_DADOS.md](CATALOGO_DADOS.md)**: Catalogo completo de dados com descricao de todas as tabelas e atributos, dominios e linhagem
- **[PIPELINE_ETL.md](PIPELINE_ETL.md)**: Documentacao detalhada do pipeline ETL com todas as transformacoes e justificativas
- **[CHECKLIST_MVP.md](CHECKLIST_MVP.md)**: Checklist de requisitos do MVP e validacao de atendimento
- **[AUTOAVALIACAO.md](AUTOAVALIACAO.md)**: Autoavaliacao final do projeto

## Resumo Executivo

Este MVP visa analisar a conformidade de ciclos de inspecao rodoviaria baseado em dados de GPS de viaturas de patrulhamento. O problema central e verificar se as viaturas estao cumprindo os requisitos contratuais de frequencia de passagem em trechos especificos da rodovia, organizados em ciclos de 3 horas ao longo do dia.

**Para detalhes completos sobre o problema e metodologia, consulte [PROBLEMA_E_METODOLOGIA.md](PROBLEMA_E_METODOLOGIA.md)**

### Perguntas de Negocio

1. Qual a taxa de conformidade por ciclo?
2. Qual a taxa de conformidade por periodo (P1/P2)?
3. Quais trechos (KMs) apresentam maior risco de nao conformidade?
4. Qual a distribuicao temporal da conformidade?
5. Quais viaturas apresentam maior regularidade?

## Estrutura do Projeto

```
mvp/
├── 00_gerar_bronze_ciclos_concatenados.ipynb  # Camada Bronze: Concatena dados brutos
├── 01_analise_qualidade_dados.ipynb           # Analise de qualidade (Bronze/Silver)
├── 02_gerar_silver_ciclos_base.ipynb          # Camada Silver Base: Tratamento inicial
├── 03_gerar_silver_ciclos_referencia.ipynb    # Camada Silver Referencia: Selecao de registros
├── 04_gerar_gold_conformidade_racional.ipynb  # Camada Gold: Agregacoes de conformidade
├── 05_gerar_painel_consolidado_mvp.ipynb      # Painel consolidado multi-KM
├── 06_analise_solucao_problema.ipynb          # Resposta as perguntas de negocio
├── dados_bronze/                              # Dados brutos concatenados (Parquet)
│   └── ciclos_concatenados.parquet
├── silver/                                    # Dados tratados (Parquet)
│   ├── silver_ciclos_base.parquet
│   └── silver_ciclos_referencia.parquet
├── gold/                                      # Dados agregados/analiticos (Parquet + Excel)
│   ├── gold_conformidade_ciclo_km.parquet
│   ├── gold_conformidade_periodo_km.parquet
│   ├── gold_conformidade_ciclo_km.xlsx
│   ├── gold_conformidade_periodo_km.xlsx
│   └── painel_consolidado_mvp_YYYYMM.xlsx
├── visualizacoes/                             # Graficos e visualizacoes geradas
│   └── [arquivos PNG]
├── README_MVP.md                              # Este arquivo
├── PROBLEMA_E_METODOLOGIA.md                  # Problema e metodologia detalhada
├── CATALOGO_DADOS.md                          # Catalogo de dados
├── PIPELINE_ETL.md                            # Documentacao do pipeline ETL
└── AUTOAVALIACAO.md                           # Autoavaliacao final
```

## Fluxo de Execucao

### Ordem Recomendada de Execucao

**Pipeline Principal (ETL)**:

1. **`00_gerar_bronze_ciclos_concatenados.ipynb`**
   - Carrega e concatena arquivos Excel brutos (`Ciclo01.xlsx`, `Ciclo02.xlsx`)
   - Gera camada Bronze: `dados_bronze/ciclos_concatenados.parquet`
   - Volume: ~1.548.566 registros

2. **`01_analise_qualidade_dados.ipynb`** (Recomendado apos Bronze)
   - Analisa qualidade dos dados brutos (Bronze)
   - Pode ser executado apos cada camada para validacao
   - Gera relatorio textual de qualidade

3. **`02_gerar_silver_ciclos_base.ipynb`**
   - Aplica transformacoes basicas (padronizacao, colunas derivadas)
   - Gera camada Silver Base: `silver/silver_ciclos_base.parquet`
   - Volume: ~1.548.566 registros

4. **`03_gerar_silver_ciclos_referencia.ipynb`**
   - Aplica filtros e seleciona registros de referencia
   - Gera camada Silver Referencia: `silver/silver_ciclos_referencia.parquet`
   - Volume: ~197.417 registros (reducao de ~87%)

5. **`04_gerar_gold_conformidade_racional.ipynb`**
   - Agrega dados para calcular conformidade por ciclo e periodo
   - Gera camada Gold: `gold/gold_conformidade_ciclo_km.parquet` e `gold_conformidade_periodo_km.parquet`
   - Tambem gera versoes Excel para visualizacao
   - Volume: ~140.960 linhas (ciclo) e ~119.117 linhas (periodo)

6. **`05_gerar_painel_consolidado_mvp.ipynb`**
   - Gera painel consolidado multi-KM em Excel
   - Seleciona top KMs por frequencia
   - Gera: `gold/painel_consolidado_mvp_YYYYMM.xlsx`

**Analises Complementares**:

7. **`06_analise_solucao_problema.ipynb`**
   - Responde as 5 perguntas de negocio
   - Gera visualizacoes e relatorios
   - Salva graficos em: `visualizacoes/`

### Notas Importantes

- **Ordem de Execucao**: Os notebooks devem ser executados na ordem numerica (00 -> 01 -> 02 -> ...)
- **Dependencias**: Cada notebook depende dos arquivos gerados pelos notebooks anteriores
- **Idempotencia**: Todos os notebooks podem ser executados multiplas vezes sem efeitos colaterais
- **Analise de Qualidade**: O notebook `01_analise_qualidade_dados` pode ser executado apos cada camada para validacao

## Arquitetura de Dados

Este MVP utiliza uma arquitetura de **Data Lake** com tres camadas:

### Bronze (Raw)
- **Fonte**: Arquivos Excel brutos (`Ciclo01.xlsx`, `Ciclo02.xlsx`)
- **Processamento**: Concatenacao vertical sem transformacoes
- **Formato**: Parquet
- **Localizacao**: `mvp/dados_bronze/ciclos_concatenados.parquet`
- **Volume**: 1.548.566 registros

### Silver (Cleaned/Transformed)
- **Silver Base**: Dados padronizados com colunas derivadas
  - Localizacao: `mvp/silver/silver_ciclos_base.parquet`
  - Transformacoes: Renomeacao de colunas, criacao de timestamp, conversao de KM, colunas derivadas (Ciclo, Minutos, etc.)
  
- **Silver Referencia**: Registros selecionados (1 por Data x Ciclo x KM)
  - Localizacao: `mvp/silver/silver_ciclos_referencia.parquet`
  - Transformacoes: Filtros previos, selecao de referencia, calculo de `Minuto_no_Ciclo`
  - Volume: ~197.417 registros

### Gold (Aggregated/Analytical)
- **Conformidade por Ciclo**: Agregacao por Mes x KM x Ciclo x Periodo
  - Localizacao: `mvp/gold/gold_conformidade_ciclo_km.parquet` e `.xlsx`
  - Metricas: `Media_Posicao_no_Ciclo`, `Saldo_Ciclo`, `Status_Ciclo`
  
- **Conformidade por Periodo**: Agregacao por Mes x KM x Periodo
  - Localizacao: `mvp/gold/gold_conformidade_periodo_km.parquet` e `.xlsx`
  - Metricas: `Saldo_Periodo`, `Status_Periodo`

- **Painel Consolidado**: Visao executiva multi-KM
  - Localizacao: `mvp/gold/painel_consolidado_mvp_YYYYMM.xlsx`
  - Formato: Excel com abas P1 e P2

**Para detalhes completos sobre transformacoes e justificativas, consulte [PIPELINE_ETL.md](PIPELINE_ETL.md)**

## Tecnologias Utilizadas

- **Python 3.10+**: Linguagem de programacao principal
- **Pandas**: Manipulacao e analise de dados
- **NumPy**: Calculos numericos e operacoes vetorizadas
- **Matplotlib**: Geracao de graficos e visualizacoes
- **Seaborn**: Visualizacoes estatisticas avancadas
- **OpenPyXL**: Leitura e escrita de arquivos Excel
- **Parquet**: Armazenamento eficiente de dados (formato colunar)
- **Excel**: Visualizacao e compartilhamento de paineis

## Regras de Negocio Implementadas

### Limites de Conformidade

- **Limite Individual (Ciclo)**: 180 minutos
  - Cada ciclo de 3 horas deve ter media de posicao <= 180 minutos
  - Avaliacao independente por ciclo

- **Limite Geral (Periodo)**: 60 minutos acumulados
  - Soma dos 4 saldos de ciclo nao pode exceder 60 minutos
  - Avaliacao por periodo (P1: ciclos 1-4, P2: ciclos 5-8)

### Selecao de Registros de Referencia

- **Criterio**: 1 registro por Data x Ciclo x Rodovia x Sentido x KM
- **Prioridade**: Registro mais proximo do fim ideal do ciclo
- **Preferencia**: Registros antes do limite (mais conservador)
- **Tolerancia**: ±15 minutos em relacao aos limites do ciclo

### Pre-processamento de KMs Alvo

- Selecao automatica dos top N KMs por frequencia de registros
- Foco em rodovia e sentido especificos (BR-153/GO, S)
- Garante representatividade estatistica adequada
- Qualquer lista de KMs pode ser utilizada (metodologia flexivel)

## Migracao para Databricks

### Status Atual

Os notebooks foram adaptados para funcionar no ambiente Databricks:

- **Caminhos**: Utilizam caminhos do workspace (`/Workspace/Users/...`)
- **Compatibilidade**: Funcionam mesmo com DBFS publico desabilitado
- **Formato**: Mantem formato Parquet para compatibilidade com Delta Lake

### Estrutura no Databricks

```
/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/
├── dados_bronze/
├── silver/
├── gold/
└── visualizacoes/
```

### Proximos Passos para Producao

1. **Upload de Dados Brutos**: Fazer upload dos arquivos Excel para DBFS/Volumes
2. **Conversao para PySpark**: Migrar operacoes Pandas para PySpark para escalabilidade
3. **Delta Lake**: Criar tabelas Delta nas camadas Bronze, Silver e Gold
4. **Databricks Workflows**: Implementar pipelines ETL agendados
5. **Dashboards**: Criar dashboards com Databricks SQL ou visualizacoes Python
6. **Monitoramento**: Implementar alertas para falhas e metricas de qualidade

### Guia de Migracao

Para instrucoes detalhadas sobre migracao para Databricks, consulte:
- [PIPELINE_ETL.md](PIPELINE_ETL.md) - Secao "Migracao para Databricks"
- Documentacao interna do projeto sobre configuracao do ambiente

## Resultados Esperados

Apos executar todos os notebooks na ordem correta, voce tera:

1. **Camada Bronze**: Dados brutos concatenados e validados
2. **Relatorio de Qualidade**: Analise completa da qualidade dos dados
3. **Camada Silver**: Dados tratados e registros de referencia selecionados
4. **Camada Gold**: Tabelas de conformidade por ciclo e periodo
5. **Painel Consolidado**: Excel com visao executiva multi-KM
6. **Analise da Solucao**: Respostas as 5 perguntas de negocio com visualizacoes

## Suporte e Documentacao

- **Documentacao Tecnica**: Consulte [PIPELINE_ETL.md](PIPELINE_ETL.md) para detalhes das transformacoes
- **Catalogo de Dados**: Consulte [CATALOGO_DADOS.md](CATALOGO_DADOS.md) para descricao completa dos dados
- **Metodologia**: Consulte [PROBLEMA_E_METODOLOGIA.md](PROBLEMA_E_METODOLOGIA.md) para base teorica
- **Validacao**: Consulte [CHECKLIST_MVP.md](CHECKLIST_MVP.md) para requisitos atendidos

## Licenca e Autoria

Este MVP foi desenvolvido como parte de um projeto academico/profissional de engenharia de dados e analise de conformidade operacional.
