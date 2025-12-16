# MVP - Analise de Conformidade de Ciclos de Atendimento

Este repositorio contem o MVP (Minimum Viable Product) desenvolvido para a disciplina de Engenharia de Dados, focado na analise de conformidade de ciclos de atendimento rodoviario utilizando Databricks.

## Estrutura do Repositorio

### Notebooks Jupyter (`.ipynb`)

O pipeline ETL e as analises estao organizados em notebooks sequenciais:

- **`00_gerar_bronze_ciclos_concatenados.ipynb`** - Camada Bronze: Carrega e concatena os dados brutos dos arquivos Excel (Ciclo01.xlsx e Ciclo02.xlsx) em um unico arquivo Parquet.

- **`01_analise_qualidade_dados.ipynb`** - Analise de Qualidade: Realiza analise exploratoria da qualidade dos dados nas camadas Bronze, Silver Base e Silver Referencia, identificando problemas e propondo solucoes.

- **`02_gerar_silver_ciclos_base.ipynb`** - Camada Silver Base: Aplica transformacoes basicas nos dados Bronze, incluindo conversao de tipos, tratamento de valores nulos e criacao de campos calculados.

- **`03_gerar_silver_ciclos_referencia.ipynb`** - Camada Silver Referencia: Seleciona os registros de referencia para cada dia e ciclo, utilizando a logica do racional antigo (selecao do registro mais proximo do fim do ciclo de 180 minutos).

- **`04_gerar_gold_conformidade_racional.ipynb`** - Camada Gold: Calcula a conformidade por ciclo e por periodo, gerando as tabelas finais de analise conforme o racional estabelecido.

- **`04_gerar_painel_consolidado_mvp.ipynb`** - Painel Consolidado: Gera o painel Excel final com analise multi-KM, apresentando medias, saldos e status de atendimento.

- **`05_analise_solucao_problema.ipynb`** - Analise de Solucao: Responde perguntas de negocio, gera visualizacoes e discute os resultados obtidos.

### Documentacao

- **`README_MVP.md`** - Documentacao detalhada do MVP, incluindo requisitos, arquitetura e instrucoes de uso.

- **`PROBLEMA_E_METODOLOGIA.md`** - Descricao do problema de negocio, metodologia de solucao e racional tecnico.

- **`CATALOGO_DADOS.md`** - Catalogo completo dos dados, incluindo descricao de atributos, dominios e linhagem de dados.

- **`PIPELINE_MVP.md`** - Documentacao detalhada do pipeline ETL, incluindo transformacoes em cada camada (Bronze, Silver, Gold).

- **`AUTOAVALIACAO.md`** - Autoavaliacao do trabalho realizado, destacando pontos fortes, desafios e aprendizados.

### Evidencias do Databricks

A pasta **`evidencias-databricks/`** contem os arquivos HTML exportados da execucao dos notebooks no ambiente Databricks. Estes arquivos contem:

- **Codigo executado**: Todas as celulas dos notebooks com seus codigos Python
- **Resultados**: Outputs das execucoes, incluindo tabelas, graficos e mensagens de log
- **Metadados**: Informacoes sobre tempo de execucao, recursos utilizados e status de cada celula

**Importante**: Os arquivos HTML precisam ser baixados do repositorio para serem visualizados corretamente, pois contem JavaScript e dados embutidos que requerem acesso local ao arquivo.

## Como os Componentes se Relacionam com o MVP

### 1. Pipeline ETL Completo

Os notebooks implementam um pipeline ETL completo seguindo a arquitetura Data Lake (Bronze -> Silver -> Gold):

- **Bronze**: Dados brutos concatenados e persistidos
- **Silver**: Dados limpos e transformados em duas etapas (Base e Referencia)
- **Gold**: Dados agregados e prontos para analise de negocio

### 2. Analise de Qualidade de Dados

O notebook `01_analise_qualidade_dados.ipynb` atende aos requisitos da disciplina de:
- Analise de qualidade em nivel de atributo
- Identificacao de problemas nos dados
- Proposta de solucoes para os problemas identificados

### 3. Solucao do Problema de Negocio

Os notebooks `04_gerar_gold_conformidade_racional.ipynb` e `05_analise_solucao_problema.ipynb` implementam:
- Calculo de conformidade conforme racional estabelecido
- Resposta a perguntas de negocio
- Visualizacoes informativas
- Discussao dos resultados

### 4. Evidencias de Execucao

Os arquivos HTML em `evidencias-databricks/` servem como:
- **Prova de execucao**: Demonstram que os notebooks foram executados com sucesso no Databricks
- **Rastreabilidade**: Permitem verificar os resultados obtidos em cada etapa
- **Documentacao executavel**: Mostram nao apenas o codigo, mas tambem os resultados reais da execucao

### 5. Documentacao Completa

A documentacao em Markdown fornece:
- Contexto do problema e metodologia
- Detalhes tecnicos do pipeline
- Catalogo de dados completo
- Autoavaliacao do trabalho

## Requisitos da Disciplina Atendidos

Este MVP atende aos seguintes requisitos da disciplina:

1. **Pipeline de Dados na Nuvem (Databricks)**
   - Implementacao completa em Databricks Community Edition
   - Uso de PySpark para processamento distribuido

2. **Arquitetura Data Lake**
   - Camadas Bronze, Silver e Gold bem definidas
   - Transformacoes documentadas em cada etapa

3. **Analise de Qualidade de Dados**
   - Analise exploratoria completa
   - Identificacao e tratamento de problemas

4. **Solucao de Problema de Negocio**
   - Implementacao do racional de conformidade
   - Resposta a perguntas de negocio
   - Visualizacoes e discussao de resultados

5. **Documentacao Completa**
   - Problema e metodologia documentados
   - Catalogo de dados detalhado
   - Pipeline ETL documentado
   - Autoavaliacao realizada

## Como Utilizar Este Repositorio

1. **Visualizar os Notebooks**: Abra os arquivos `.ipynb` em um ambiente Jupyter ou no Databricks
2. **Consultar a Documentacao**: Leia os arquivos `.md` para entender o contexto e a metodologia
3. **Verificar as Evidencias**: Baixe os arquivos HTML da pasta `evidencias-databricks/` e abra-os em um navegador para ver os resultados das execucoes

## Tecnologias Utilizadas

- **Databricks Community Edition**: Plataforma de processamento de dados na nuvem
- **PySpark**: Processamento distribuido de dados
- **Python**: Linguagem de programacao principal
- **Pandas**: Manipulacao de dados
- **Matplotlib/Seaborn**: Visualizacao de dados
- **Parquet**: Formato de armazenamento de dados
- **Excel**: Geracao de paineis consolidados

## Estrutura de Dados

Os dados seguem o seguinte fluxo:

```
Excel (Ciclo01.xlsx, Ciclo02.xlsx)
    ↓
Bronze (ciclos_concatenados.parquet)
    ↓
Silver Base (silver_ciclos_base.parquet)
    ↓
Silver Referencia (silver_ciclos_referencia.parquet)
    ↓
Gold (gold_conformidade_ciclo_km.parquet, gold_conformidade_periodo_km.parquet)
    ↓
Painel Consolidado (Excel)
```

## Contato e Suporte

Para duvidas ou sugestoes sobre este MVP, consulte a documentacao detalhada nos arquivos `.md` ou verifique as evidencias de execucao nos arquivos HTML.

---

**Desenvolvido como parte do MVP da disciplina de Engenharia de Dados**

