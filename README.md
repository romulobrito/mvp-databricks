# MVP - Análise de Conformidade de Ciclos de Atendimento

Este repositório contém o MVP (Minimum Viable Product) desenvolvido para a disciplina de Engenharia de Dados, focado na análise de conformidade de ciclos de atendimento rodoviário utilizando Databricks.

## Estrutura do Repositório

### Notebooks Jupyter (`.ipynb`)

O pipeline ETL e as análises estão organizados em notebooks sequenciais:

- **`00_gerar_bronze_ciclos_concatenados.ipynb`** - Camada Bronze: Carrega e concatena os dados brutos dos arquivos Excel (Ciclo01.xlsx e Ciclo02.xlsx) em um único arquivo Parquet.

- **`01_analise_qualidade_dados.ipynb`** - Análise de Qualidade: Realiza análise exploratória da qualidade dos dados nas camadas Bronze, Silver Base e Silver Referência, identificando problemas e propondo soluções.

- **`02_gerar_silver_ciclos_base.ipynb`** - Camada Silver Base: Aplica transformações básicas nos dados Bronze, incluindo conversão de tipos de dados, tratamento de valores nulos, padronização de formatos de data/hora, criação de campos calculados (como minutos desde o início do dia e identificação de ciclos), e aplicação de filtros iniciais para garantir a qualidade dos dados antes das próximas etapas do pipeline.

- **`03_gerar_silver_ciclos_referencia.ipynb`** - Camada Silver Referência: Seleciona os registros de referência para cada dia e ciclo, utilizando a lógica do racional antigo. Para cada combinação de Data e Ciclo, identifica o registro mais próximo do fim ideal do ciclo (180 minutos), priorizando registros antes ou no limite do ciclo. Este processo garante que apenas um registro representativo seja mantido por dia e ciclo, preparando os dados para o cálculo de conformidade na camada Gold.

- **`04_gerar_gold_conformidade_racional.ipynb`** - Camada Gold: Calcula a conformidade por ciclo e por período, gerando as tabelas finais de análise conforme o racional estabelecido.

- **`04_gerar_painel_consolidado_mvp.ipynb`** - Painel Consolidado: Gera o painel Excel final com análise multi-KM, apresentando médias, saldos e status de atendimento.

- **`05_analise_solucao_problema.ipynb`** - Análise de Solução: Responde perguntas de negócio, gera visualizações e discute os resultados obtidos.

### Documentação

- **`README_MVP.md`** - Documentação detalhada do MVP, incluindo requisitos, arquitetura e instruções de uso.

- **`PROBLEMA_E_METODOLOGIA.md`** - Descrição do problema de negócio, metodologia de solução e racional técnico.

- **`CATALOGO_DADOS.md`** - Catálogo completo dos dados, incluindo descrição de atributos, domínios e linhagem de dados.

- **`PIPELINE_MVP.md`** - Documentação detalhada do pipeline ETL, incluindo transformações em cada camada (Bronze, Silver, Gold).

- **`AUTOAVALIACAO.md`** - Autoavaliação do trabalho realizado, destacando pontos fortes, desafios e aprendizados.

### Evidências do Databricks

A pasta **`evidencias-databricks/`** contém os arquivos HTML exportados da execução dos notebooks no ambiente Databricks. Estes arquivos contêm:

- **Código executado**: Todas as células dos notebooks com seus códigos Python
- **Resultados**: Outputs das execuções, incluindo tabelas, gráficos e mensagens de log
- **Metadados**: Informações sobre tempo de execução, recursos utilizados e status de cada célula

**⚠️ IMPORTANTE - Como Visualizar as Evidências**: 

Os arquivos HTML **NÃO podem ser visualizados diretamente no GitHub**. É **necessário baixar** os arquivos HTML da pasta `evidencias-databricks/` para o seu computador local e abri-los em um navegador web (Chrome, Firefox, etc.) para visualizar corretamente. Isso é necessário porque os arquivos contêm JavaScript e dados embutidos que requerem acesso local ao arquivo para funcionar adequadamente.

## Como os Componentes se Relacionam com o MVP

### 1. Pipeline ETL Completo

Os notebooks implementam um pipeline ETL completo seguindo a arquitetura Data Lake (Bronze -> Silver -> Gold):

- **Bronze**: Dados brutos concatenados e persistidos
- **Silver**: Dados limpos e transformados em duas etapas (Base e Referencia)
- **Gold**: Dados agregados e prontos para analise de negocio

### 2. Análise de Qualidade de Dados

O notebook `01_analise_qualidade_dados.ipynb` atende aos requisitos da disciplina de:
- Análise de qualidade em nível de atributo
- Identificação de problemas nos dados
- Proposta de soluções para os problemas identificados

### 3. Solução do Problema de Negócio

Os notebooks `04_gerar_gold_conformidade_racional.ipynb` e `05_analise_solucao_problema.ipynb` implementam:
- Cálculo de conformidade conforme racional estabelecido
- Resposta a perguntas de negócio
- Visualizações informativas
- Discussão dos resultados

### 4. Evidências de Execução

Os arquivos HTML em `evidencias-databricks/` servem como:
- **Prova de execução**: Demonstram que os notebooks foram executados com sucesso no Databricks
- **Rastreabilidade**: Permitem verificar os resultados obtidos em cada etapa
- **Documentação executável**: Mostram não apenas o código, mas também os resultados reais da execução

### 5. Documentação Completa

A documentação em Markdown fornece:
- Contexto do problema e metodologia
- Detalhes técnicos do pipeline
- Catálogo de dados completo
- Autoavaliação do trabalho

## Requisitos da Disciplina Atendidos

Este MVP atende aos seguintes requisitos da disciplina:

1. **Pipeline de Dados na Nuvem (Databricks)**
   - Implementação completa em Databricks Community Edition
   - Uso de PySpark para processamento distribuído

2. **Arquitetura Data Lake**
   - Camadas Bronze, Silver e Gold bem definidas
   - Transformações documentadas em cada etapa

3. **Análise de Qualidade de Dados**
   - Análise exploratória completa
   - Identificação e tratamento de problemas

4. **Solução de Problema de Negócio**
   - Implementação do racional de conformidade
   - Resposta a perguntas de negócio
   - Visualizações e discussão de resultados

5. **Documentação Completa**
   - Problema e metodologia documentados
   - Catálogo de dados detalhado
   - Pipeline ETL documentado
   - Autoavaliação realizada

## Como Utilizar Este Repositorio

1. **Visualizar os Notebooks**: Abra os arquivos `.ipynb` em um ambiente Jupyter ou no Databricks
2. **Consultar a Documentacao**: Leia os arquivos `.md` para entender o contexto e a metodologia
3. **Verificar as Evidencias**: Baixe os arquivos HTML da pasta `evidencias-databricks/` e abra-os em um navegador para ver os resultados das execucoes

## Tecnologias Utilizadas

- **Databricks Community Edition**: Plataforma de processamento de dados na nuvem
- **PySpark**: Processamento distribuído de dados
- **Python**: Linguagem de programação principal
- **Pandas**: Manipulação de dados
- **Matplotlib/Seaborn**: Visualização de dados
- **Parquet**: Formato de armazenamento de dados
- **Excel**: Geração de painéis consolidados

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

Para dúvidas ou sugestões sobre este MVP, consulte a documentação detalhada nos arquivos `.md` ou verifique as evidências de execução nos arquivos HTML.

---

**Desenvolvido como parte do MVP da disciplina de Engenharia de Dados**

