# Documentacao do Pipeline ETL - MVP

## Visao Geral

Este documento descreve os processos de **Extracao, Transformacao e Carga (ETL)** implementados no MVP, detalhando cada etapa de transformacao e as tecnicas utilizadas para processar dados de GPS de viaturas e avaliar conformidade de frequencia de passagem em trechos rodoviarios.

## Arquitetura do Pipeline

O pipeline segue uma arquitetura de **Data Lake** com tres camadas:

```
Bronze (Raw) -> Silver (Cleaned/Transformed) -> Gold (Aggregated/Analytical)
```

### Estrutura de Diretorios no Databricks

```
/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/
├── dados_bronze/
│   └── ciclos_concatenados.parquet
├── silver/
│   ├── silver_ciclos_base.parquet
│   └── silver_ciclos_referencia.parquet
├── gold/
│   ├── gold_conformidade_ciclo_km.parquet
│   ├── gold_conformidade_periodo_km.parquet
│   ├── gold_conformidade_ciclo_km.xlsx
│   ├── gold_conformidade_periodo_km.xlsx
│   └── painel_consolidado_mvp_YYYYMM.xlsx
└── visualizacoes/
    └── [arquivos PNG gerados]
```

## Etapa 0: Bronze - Concatenacao de Dados Brutos

### Notebook: `00_gerar_bronze_ciclos_concatenados.ipynb`

#### Objetivo

Carregar e concatenar arquivos Excel brutos contendo dados de GPS de viaturas, criando a camada Bronze do Data Lake.

#### Extracao

- **Fonte**: Arquivos Excel (`Ciclo01.xlsx`, `Ciclo02.xlsx`)
- **Localizacao**: 
  - `/dbfs/Workspace/Users/romulobrtsilva@gmail.com/Drafts/raw_data/`
  - `/dbfs/FileStore/shared_uploads/`
  - `/dbfs/FileStore/uploads/`
- **Metodo**: `pandas.read_excel()` com `openpyxl`
- **Volume**: 1.548.566 registros totais apos concatenacao
- **Schema Original**:
  - `DataEv`: Data do evento (datetime)
  - `HoraEv`: Hora do evento (string HH:MM:SS)
  - `CodRecurso`: Codigo numerico do recurso/viatura
  - `Recursos`: Nome/identificacao da viatura
  - `Latitude`: Coordenada geografica latitude
  - `Longitude`: Coordenada geografica longitude
  - `SiglaRodovia`: Sigla da rodovia (ex: "BR-153/GO")
  - `Sentido`: Sentido da rodovia ("N" ou "S")
  - `KM`: Quilometragem no formato "XXX+YYY" (string)

#### Transformacao

**1. Concatenacao Vertical**:
```python
df_concatenado = pd.concat([df_ciclo01, df_ciclo02], ignore_index=True)
```

**Justificativa**: 
- Unir os dois arquivos Excel em um unico DataFrame
- `ignore_index=True` garante indices sequenciais unicos
- Nao ha necessidade de conciliacao pois os arquivos tem o mesmo schema

**2. Validacao de Dados**:
- Verificacao de existencia dos arquivos fonte
- Contagem de linhas antes e apos concatenacao
- Exibicao de resumo das colunas disponiveis
- Verificacao de tipos de dados

**Sem Transformacoes Adicionais**: 
- Dados mantidos em formato bruto (sem limpeza ou padronizacao)
- Todas as colunas originais preservadas
- Valores nulos mantidos como estao

#### Carga

- **Destino Principal**: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/dados_bronze/ciclos_concatenados.parquet`
- **Destinos Alternativos** (caso DBFS publico esteja desabilitado):
  - `/dbfs/FileStore/shared_uploads/mvp/dados_bronze/ciclos_concatenados.parquet`
  - `/dbfs/FileStore/mvp/dados_bronze/ciclos_concatenados.parquet`
- **Formato**: Parquet (compressao eficiente, preserva tipos de dados)
- **Modo**: `overwrite` (idempotente)
- **Volume Final**: 1.548.566 registros

#### Documentacao de Transformacao

**Juncao de Conjuntos de Dados**:

Os dois arquivos Excel (`Ciclo01.xlsx` e `Ciclo02.xlsx`) sao concatenados verticalmente, mantendo todas as colunas. Nao ha necessidade de conciliacao porque:

1. **Schema Identico**: Ambos os arquivos possuem exatamente as mesmas colunas e tipos de dados
2. **Sem Chaves Primarias**: Nao existem chaves primarias que precisem ser reconciliadas ou deduplicadas
3. **Dados Complementares**: Os arquivos representam dados de diferentes periodos ou fontes que se complementam
4. **Sem Sobreposicao**: Nao ha registros duplicados entre os arquivos que precisem ser tratados

**Validacao Implementada**:

- Verificacao de existencia dos arquivos fonte em multiplos caminhos possiveis
- Contagem de linhas antes e apos concatenacao
- Exibicao de resumo das colunas disponiveis e seus tipos
- Verificacao de tamanho do arquivo gerado

**Tratamento de Erros**:

- Tentativa de leitura de multiplos caminhos possiveis para os arquivos Excel
- Mensagens de erro claras caso os arquivos nao sejam encontrados
- Verificacao de escrita bem-sucedida do arquivo Parquet

## Etapa 1: Analise de Qualidade de Dados

### Notebook: `01_analise_qualidade_dados.ipynb`

#### Objetivo

Realizar analise de qualidade para cada atributo do conjunto de dados, identificando problemas e propondo solucoes antes de processar os dados nas camadas seguintes.

#### Extracao

- **Fontes**: 
  - `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/dados_bronze/ciclos_concatenados.parquet`
  - `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/silver/silver_ciclos_base.parquet` (apos ser gerado)
  - `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/silver/silver_ciclos_referencia.parquet` (apos ser gerado)
- **Metodo**: `pandas.read_parquet()`
- **Volume**: Varia conforme a camada analisada

#### Transformacoes Aplicadas

**1. Analise da Camada Bronze**:

```python
# Completude (Valores Nulos)
nulos_bronze = df_bronze.isnull().sum()
pct_nulos = (nulos_bronze / len(df_bronze)) * 100

# Consistencia de Tipos
print(df_bronze.dtypes)

# Valores Unicos por Coluna Categorica
print(df_bronze["SiglaRodovia"].unique())
print(df_bronze["Sentido"].unique())

# Faixas de Valores Numericos
print(f"CodRecurso: Min={df_bronze['CodRecurso'].min()}, Max={df_bronze['CodRecurso'].max()}")
print(f"Latitude: Min={df_bronze['Latitude'].min()}, Max={df_bronze['Latitude'].max()}")

# Duplicatas Completas
duplicatas = df_bronze.duplicated().sum()
```

**Justificativa**: 
- Verificar completude dos dados (valores nulos)
- Validar consistencia de tipos de dados
- Identificar valores unicos e faixas de valores para validacao de dominio
- Detectar registros completamente duplicados

**2. Analise da Camada Silver Base**:

```python
# Completude apos Transformacoes
nulos_silver = df_silver_base.isnull().sum()

# Validacao de Dominios
ciclos_invalidos = df_silver_base[(df_silver_base['Ciclo'] < 1) | (df_silver_base['Ciclo'] > 8)]
minutos_invalidos = df_silver_base[(df_silver_base['Minutos'] < 0) | (df_silver_base['Minutos'] >= 1440)]
km_invalidos = df_silver_base[df_silver_base['KM'] <= 0]

# Estatisticas Descritivas
print(df_silver_base[["Ciclo", "Minutos", "KM", "Inicio_Ideal", "Fim_Ideal"]].describe())
```

**Justificativa**: 
- Verificar se as transformacoes introduziram valores nulos
- Validar que valores estao dentro dos dominios esperados:
  - Ciclo: [1, 8]
  - Minutos: [0, 1440)
  - KM: > 0
- Obter estatisticas descritivas para identificar outliers ou distribuicoes anormais

**3. Analise da Camada Silver Referencia**:

```python
# Validacao de Minuto_no_Ciclo
print(f"Min: {df_silver_ref['Minuto_no_Ciclo'].min():.2f}")
print(f"Max: {df_silver_ref['Minuto_no_Ciclo'].max():.2f}")
print(f"Media: {df_silver_ref['Minuto_no_Ciclo'].mean():.2f}")

# Valores fora do dominio [0, 180)
fora_dominio = df_silver_ref[(df_silver_ref["Minuto_no_Ciclo"] < 0) | 
                              (df_silver_ref["Minuto_no_Ciclo"] >= 180)]

# Distribuicao por Ciclo
dist_ciclo = df_silver_ref.groupby("Ciclo")["Minuto_no_Ciclo"].agg(["count", "mean", "std"])

# Distribuicao por Rodovia e Sentido
dist_rod_sent = df_silver_ref.groupby(["Rodovia", "Sentido"]).size()
```

**Justificativa**: 
- Validar que `Minuto_no_Ciclo` esta dentro do dominio esperado [0, 180)
- Analisar distribuicao por ciclo para identificar padroes
- Verificar cobertura por rodovia e sentido

**4. Resumo de Problemas e Solucoes**:

```python
# Consolidacao de problemas encontrados
problemas_consolidados = []

# Problemas da camada Bronze
if duplicatas > 0:
    problemas_consolidados.append({
        "Camada": "Bronze",
        "Atributo": "Todos",
        "Problema": "Duplicatas completas",
        "Quantidade": duplicatas,
        "Impacto nas Perguntas": "Pode afetar contagens e agregacoes",
        "Solucao": "Remover duplicatas antes do processamento",
        "Status": "Tratado"
    })

# Problemas da camada Silver Base
# ... (adicionar problemas encontrados)

# Problemas da camada Silver Referencia
if len(fora_dominio) > 0:
    problemas_consolidados.append({
        "Camada": "Silver Referencia",
        "Atributo": "Minuto_no_Ciclo",
        "Problema": "Valores fora do dominio [0, 180)",
        "Quantidade": len(fora_dominio),
        "Impacto nas Perguntas": "Pode afetar avaliacao de conformidade",
        "Solucao": "Verificar logica de calculo ou aplicar filtros de tolerancia",
        "Status": "A verificar"
    })
```

**Justificativa**: 
- Consolidar todos os problemas encontrados em uma estrutura unificada
- Avaliar impacto nas perguntas de negocio
- Propor solucoes para cada problema identificado
- Rastrear status de tratamento

#### Carga

- **Destino**: Relatorio textual exibido no notebook (nao gera arquivo)
- **Formato**: Saida de texto com tabelas e estatisticas
- **Conteudo**:
  - Resumo de completude por camada
  - Validacao de dominios
  - Estatisticas descritivas
  - Lista de problemas identificados
  - Propostas de solucao

#### Documentacao de Transformacao

**Dimensoes de Qualidade Analisadas**:

1. **Completude**: 
   - Verificacao de valores nulos por atributo
   - Percentual de completude por coluna
   - Identificacao de atributos com problemas de completude

2. **Consistencia**: 
   - Validacao de tipos de dados
   - Verificacao de valores dentro dos dominios esperados
   - Identificacao de valores invalidos ou fora do dominio

3. **Integridade**: 
   - Deteccao de duplicatas completas
   - Verificacao de integridade referencial entre camadas
   - Validacao de chaves primarias/combinacoes unicas

4. **Precisao**: 
   - Validacao de formatos (datas, horas, KM)
   - Identificacao de outliers significativos
   - Verificacao de valores dentro de faixas esperadas

**Validacao Implementada**:

- **Camada Bronze**:
  - Completude de todos os atributos
  - Consistencia de tipos
  - Valores unicos por coluna categorica
  - Faixas de valores numericos
  - Duplicatas completas

- **Camada Silver Base**:
  - Completude apos transformacoes
  - Validacao de dominios (Ciclo, Minutos, KM)
  - Estatisticas descritivas de colunas numericas

- **Camada Silver Referencia**:
  - Validacao de `Minuto_no_Ciclo` (dominio [0, 180))
  - Distribuicao por ciclo
  - Distribuicao por rodovia e sentido

**Impacto nas Perguntas de Negocio**:

A analise de qualidade identifica problemas que podem afetar as respostas das perguntas de negocio:

- **Pergunta 1 (Taxa de conformidade por ciclo)**: Depende de Ciclo e Minuto_no_Ciclo validos
- **Pergunta 2 (Taxa de conformidade por periodo)**: Depende de Periodo e Saldo_Periodo calculados corretamente
- **Pergunta 3 (Trechos com maior risco)**: Depende de KM valido e valores nao nulos
- **Pergunta 4 (Distribuicao temporal)**: Depende de Ciclo, Periodo e Minuto_no_Ciclo validos
- **Pergunta 5 (Regularidade por viatura)**: Depende de Recurso e Minuto_no_Ciclo validos

**Conclusao da Analise**:

- Resumo por dimensao de qualidade (Completude, Consistencia, Integridade, Precisao)
- Status geral da qualidade dos dados
- Recomendacoes para tratamento de problemas identificados
- Validacao de que os dados estao prontos para analise de conformidade

## Etapa 2: Silver Base - Tratamento Inicial

### Notebook: `02_gerar_silver_ciclos_base.ipynb`

#### Objetivo

Aplicar transformacoes basicas nos dados brutos da camada Bronze, padronizando formatos e criando colunas derivadas necessarias para analise de conformidade de ciclos.

#### Extracao

- **Fonte**: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/dados_bronze/ciclos_concatenados.parquet`
- **Caminhos Alternativos** (para compatibilidade):
  - `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/dados_bronze/dados_bronze/ciclos_concatenados.parquet`
- **Metodo**: `pandas.read_parquet()`
- **Volume**: 1.548.566 registros

#### Transformacoes Aplicadas

**1. Renomeacao de Colunas (Padronizacao)**:
```python
mapeamento_colunas = {
    "DataEv": "DataOcorrencia",
    "HoraEv": "HoraAcionamento",
    "CodRecurso": "CodigoRecurso",
    "Recursos": "Recurso",
    "SiglaRodovia": "Rodovia",
}
df = df_raw.rename(columns=mapeamento_colunas).copy()
```

**Justificativa**: 
- Padronizar nomes de colunas para facilitar processamento posterior
- Tornar nomes mais descritivos e consistentes
- Facilitar manutencao e leitura do codigo

**2. Criacao de Timestamp (`Marcacao`)**:
```python
df["Marcacao"] = pd.to_datetime(
    df["DataOcorrencia"].astype(str) + " " + df["HoraAcionamento"].astype(str),
    errors="coerce"
)
```

**Justificativa**: 
- Combinar data e hora em um unico campo timestamp para facilitar calculos temporais
- Permitir operacoes de ordenacao, filtragem e agregacao temporal
- `errors="coerce"` converte valores invalidos para `NaT` (Not a Time) em vez de gerar erro

**3. Conversao de KM (String para Float)**:
```python
def converter_km(km_str):
    """Converte formato XXX+YYY para float (XXX.YYY)."""
    if pd.isna(km_str) or str(km_str).strip() == "":
        return np.nan
    s = str(km_str)
    if "+" in s:
        partes = s.split("+")
        if len(partes) == 2 and partes[0].strip() != "" and partes[1].strip() != "":
            try:
                return float(partes[0]) + float(partes[1]) / 1000.0
            except ValueError:
                return np.nan
    try:
        return float(s.replace("+", ""))
    except ValueError:
        return np.nan

df["KM_original"] = df.get("KM", np.nan)
df["KM"] = df["KM"].apply(converter_km)
```

**Justificativa**: 
- Converter formato de quilometragem brasileiro padrao (XXX+YYY) para formato numerico decimal (XXX.YYY)
- Exemplo: "87+260" -> 87.260
- Preservar valor original em `KM_original` para rastreabilidade
- Tratar casos especiais (valores nulos, formatos invalidos)

**4. Remocao de Registros Invalidos**:
```python
antes = len(df)
df = df[df["Marcacao"].notna()].copy()
depois = len(df)
print(f"Registros com Marcacao valida: {depois:,} (removidos {antes - depois:,})")
```

**Justificativa**: 
- Remover registros sem timestamp valido (erros de parse de data/hora)
- Garantir que todos os registros tenham informacao temporal valida para analise
- Registrar quantidade de registros removidos para auditoria

**5. Criacao de Colunas Derivadas**:
```python
# Data (extraida do timestamp)
df["Data"] = df["Marcacao"].dt.date

# Ciclo (ciclos de 3 horas: 1=00-03h, 2=03-06h, ..., 8=21-24h)
df["Ciclo"] = df["Marcacao"].dt.hour // 3 + 1

# Minutos desde 00:00
df["Minutos"] = (
    df["Marcacao"].dt.hour * 60
    + df["Marcacao"].dt.minute
    + df["Marcacao"].dt.second / 60.0
)

# Limites ideais do ciclo (em minutos desde 00:00)
df["Inicio_Ideal"] = (df["Ciclo"] - 1) * 180
df["Fim_Ideal"] = df["Ciclo"] * 180
```

**Justificativa**: 
- `Data`: Extrair apenas a data para agrupamentos diarios
- `Ciclo`: Dividir o dia em 8 ciclos de 3 horas cada (requisito contratual)
- `Minutos`: Converter timestamp para minutos desde meia-noite para calculos de posicao no ciclo
- `Inicio_Ideal` e `Fim_Ideal`: Definir limites temporais de cada ciclo para avaliacao de conformidade

#### Carga

- **Destino**: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/silver/silver_ciclos_base.parquet`
- **Selecao de Colunas**: Apenas colunas relevantes para analise
  - `Marcacao`, `Data`, `Rodovia`, `Sentido`, `KM`, `KM_original`
  - `Recurso`, `CodigoRecurso`
  - `Ciclo`, `Minutos`, `Inicio_Ideal`, `Fim_Ideal`
- **Volume**: 1.548.566 registros (mantido apos remocao de invalidos)
- **Tamanho**: ~26.33 MB

#### Documentacao de Transformacao

**Tratamento de Dados Invalidos**:

- **Registros com `Marcacao` Invalida**: Removidos (0 registros no dataset atual)
- **KM Invalido**: Convertido para `NaN` e preservado para analise posterior
- **Valores Nulos**: Mantidos como estao (nao removidos preventivamente)

**Validacao Implementada**:

- Verificacao de existencia do arquivo Bronze em multiplos caminhos
- Contagem de registros antes e apos filtros
- Exibicao de resumo estatistico:
  - Periodo coberto (data minima e maxima)
  - Rodovias presentes
  - Sentidos presentes
  - Faixa de KM (minimo e maximo)
- Verificacao de tamanho do arquivo gerado

**Preservacao de Dados Originais**:

- `KM_original`: Preserva formato original do KM para rastreabilidade
- Todas as colunas originais relevantes mantidas
- Timestamp original preservado em `Marcacao`

## Etapa 3: Silver Referencia - Selecao de Registros

### Notebook: `03_gerar_silver_ciclos_referencia.ipynb`

#### Objetivo

Aplicar filtros e selecionar um registro de referencia por combinacao de chaves, reduzindo volume de dados e preparando para agregacoes de conformidade.

#### Extracao

- **Fonte**: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/silver/silver_ciclos_base.parquet`
- **Metodo**: `pandas.read_parquet()`
- **Volume Inicial**: 1.548.566 registros

#### Transformacoes Aplicadas

**1. Filtros Previos (Reducao de Volume)**:

```python
# Filtro por rodovia
RODOVIA_TARGET = "BR-153/GO"
if RODOVIA_TARGET:
    df = df[df["Rodovia"] == RODOVIA_TARGET].copy()

# Filtro por sentido
SENTIDO_TARGET = "S"
if SENTIDO_TARGET:
    df = df[df["Sentido"] == SENTIDO_TARGET].copy()

# Filtro de tolerancia de tempo
TOLERANCIA_TEMPO_ANTES = 15   # minutos antes do inicio do ciclo
TOLERANCIA_TEMPO_DEPOIS = 15  # minutos depois do fim do ciclo

df["Ini_OK"] = df["Inicio_Ideal"] - TOLERANCIA_TEMPO_ANTES
df["Fim_OK"] = df["Fim_Ideal"] + TOLERANCIA_TEMPO_DEPOIS
df = df[(df["Minutos"] >= df["Ini_OK"]) & (df["Minutos"] < df["Fim_OK"])].copy()
```

**Justificativa**: 
- Reduzir volume de dados antes da selecao de referencia
- Focar em rodovia/sentido especificos conforme requisitos do negocio
- Remover registros muito distantes do ciclo (fora da janela de tolerancia)
- Tolerancia de ±15 minutos permite capturar passagens proximas aos limites do ciclo

**2. Logica de Selecao de Referencia**:

```python
# Preferencia: 0 se passou antes (ou exatamente) do Fim_Ideal, 1 se apos
df["Preferencia"] = np.where(df["Minutos"] <= df["Fim_Ideal"], 0, 1)

# Delta: distancia ate o fim ideal do ciclo
df["Delta_limite"] = np.where(
    df["Preferencia"] == 0,
    df["Fim_Ideal"] - df["Minutos"],      # Distancia antes do limite
    df["Minutos"] - df["Fim_Ideal"]       # Distancia apos o limite
)

# Ordenacao para selecao deterministica
colunas_ordenacao = ["Data", "Ciclo", "Rodovia", "Sentido", "KM", 
                     "Preferencia", "Delta_limite", "Marcacao"]

df_sorted = df.sort_values(colunas_ordenacao).copy()

# Selecao: 1 registro por combinacao de chaves
chaves = ["Data", "Ciclo", "Rodovia", "Sentido", "KM"]

df_ref = (
    df_sorted
    .groupby(chaves, as_index=False)
    .first()
)
```

**Justificativa**: 
- Selecionar o registro mais representativo de cada combinacao
- Priorizar registros proximos ao fim ideal do ciclo (mais conservador)
- Preferir registros antes do limite (preferencia = 0) sobre apos o limite
- Ordenacao deterministica garante reprodutibilidade
- `groupby().first()` preserva o primeiro registro da ordenacao

**3. Calculo de Posicao no Ciclo**:

```python
df_ref["Minuto_no_Ciclo"] = df_ref["Minutos"] - df_ref["Inicio_Ideal"]
```

**Justificativa**: 
- Calcular posicao dentro do ciclo (0-180 minutos) para avaliacao de conformidade
- Valor 0 = inicio do ciclo, valor 180 = fim do ciclo
- Necessario para calcular media mensal e avaliar se esta dentro do limite de 180 minutos

#### Carga

- **Destino**: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/silver/silver_ciclos_referencia.parquet`
- **Volume**: ~197.417 registros (reducao significativa de ~87%)
- **Tamanho**: ~3.5 MB

#### Documentacao de Transformacao

**Conciliacao de Dados**:

A selecao de referencia resolve o problema de **multiplicidade de registros** para a mesma combinacao de chaves. A logica garante:

1. **Consistencia**: Sempre seleciona o mesmo registro para a mesma combinacao (ordenacao deterministica)
2. **Representatividade**: Escolhe o registro mais proximo do fim ideal do ciclo
3. **Preferencia Conservadora**: Prioriza registros antes do limite (mais proximo do fim sem ultrapassar)
4. **Reprodutibilidade**: Ordenacao fixa garante resultados identicos em execucoes repetidas

**Reducao de Volume**:

- **Antes dos Filtros**: 1.548.566 registros
- **Apos Filtro Rodovia**: Reducao significativa
- **Apos Filtro Sentido**: Reducao adicional
- **Apos Filtro Tolerancia**: Reducao adicional
- **Apos Selecao de Referencia**: ~197.417 registros (reducao de ~87%)

**Validacao Implementada**:

- Contagem de registros apos cada filtro
- Estatisticas descritivas de `Minuto_no_Ciclo`:
  - Minimo, maximo, media, mediana
  - Verificacao de valores dentro do dominio [0, 180)
- Distribuicao por ciclo
- Distribuicao por rodovia e sentido
- Verificacao de existencia de colunas necessarias

**Colunas Preservadas**:

- Todas as colunas da Silver Base
- `Preferencia`: Indica se registro esta antes (0) ou apos (1) do fim ideal
- `Delta_limite`: Distancia ate o fim ideal do ciclo
- `Minuto_no_Ciclo`: Posicao dentro do ciclo (0-180 min)

## Etapa 4: Gold - Agregacao de Conformidade

### Notebook: `04_gerar_gold_conformidade_racional.ipynb`

#### Objetivo

Agregar dados da camada Silver Referencia para calcular metricas de conformidade por ciclo e por periodo, criando tabelas analiticas prontas para visualizacao e tomada de decisao.

#### Extracao

- **Fonte**: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/silver/silver_ciclos_referencia.parquet`
- **Metodo**: `pandas.read_parquet()`
- **Volume**: ~197.417 registros

#### Transformacoes Aplicadas

**1. Criacao de Colunas de Agregacao**:

```python
# Mes no formato AAAA-MM
df_ref["Mes"] = pd.to_datetime(df_ref["Data"]).dt.to_period("M").astype(str)

# Periodo: P1 (ciclos 1-4) ou P2 (ciclos 5-8)
df_ref["Periodo"] = np.where(df_ref["Ciclo"] <= 4, "P1", "P2")
```

**Justificativa**: 
- `Mes`: Agrupar dados por mes para analise mensal de conformidade
- `Periodo`: Dividir o dia em dois periodos conforme requisito contratual
  - P1: Ciclos 1-4 (00:00-12:00)
  - P2: Ciclos 5-8 (12:00-24:00)

**2. Agregacao por Ciclo**:

```python
# Agrupamento por Mes x KM x Ciclo x Periodo
gb_cols = ["Mes", "KM", "Ciclo", "Periodo"]

df_ciclo = (
    df_ref
    .groupby(gb_cols, as_index=False)
    .agg(Media_Posicao_no_Ciclo=("Minuto_no_Ciclo", "mean"))
)

# Calculo de saldo e status
LIMITE_INDIVIDUAL = 180.0  # minutos por ciclo

df_ciclo["Saldo_Ciclo"] = df_ciclo["Media_Posicao_no_Ciclo"] - LIMITE_INDIVIDUAL

df_ciclo["Status_Ciclo"] = np.where(
    df_ciclo["Media_Posicao_no_Ciclo"] <= LIMITE_INDIVIDUAL,
    "CONFORME",
    "NAO CONFORME",
)
```

**Justificativa**: 
- Calcular media mensal da posicao no ciclo para cada combinacao de KM e ciclo
- `Media_Posicao_no_Ciclo`: Media de todos os registros selecionados no mes para aquele KM e ciclo
- `Saldo_Ciclo`: Diferenca entre media e limite (negativo = margem, positivo = excedente)
- `Status_Ciclo`: Avaliacao binaria de conformidade (<= 180 min = CONFORME)

**3. Agregacao por Periodo**:

```python
# Agrupamento por Mes x KM x Periodo
gb_periodo = ["Mes", "KM", "Periodo"]

df_periodo = (
    df_ciclo
    .groupby(gb_periodo, as_index=False)
    .agg(Saldo_Periodo=("Saldo_Ciclo", "sum"))
)

# Calculo de status do periodo
LIMITE_GERAL = 60.0  # minutos acumulados por periodo

df_periodo["Status_Periodo"] = np.where(
    df_periodo["Saldo_Periodo"] <= LIMITE_GERAL,
    "CONFORME",
    "NAO CONFORME",
)
```

**Justificativa**: 
- Calcular saldo acumulado do periodo (soma dos 4 saldos de ciclo)
- `Saldo_Periodo`: Soma dos saldos dos 4 ciclos do periodo
- `Status_Periodo`: Avaliacao binaria de conformidade geral (<= 60 min acumulados = CONFORME)
- Regra de negocio: Mesmo que cada ciclo individual seja conforme, o periodo pode ser nao conforme se a soma dos saldos exceder 60 minutos

#### Carga

- **Destinos**:
  - Parquet: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/gold/gold_conformidade_ciclo_km.parquet`
  - Parquet: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/gold/gold_conformidade_periodo_km.parquet`
  - Excel: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/gold/gold_conformidade_ciclo_km.xlsx`
  - Excel: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/gold/gold_conformidade_periodo_km.xlsx`
- **Volume**:
  - Tabela Ciclo: ~140.960 linhas
  - Tabela Periodo: ~119.117 linhas

#### Documentacao de Transformacao

**Agregacoes Hierarquicas**:

1. **Nivel Ciclo**: Agrega por `Mes x KM x Ciclo x Periodo`
   - Calcula media de `Minuto_no_Ciclo` (media mensal)
   - Avalia conformidade individual (<= 180 min)
   - Calcula saldo (diferenca entre media e limite)

2. **Nivel Periodo**: Agrega por `Mes x KM x Periodo`
   - Soma os 4 saldos de ciclo do periodo
   - Avalia conformidade geral (<= 60 min acumulados)
   - Mantem granularidade por KM para analise espacial

**Regras de Negocio**:

- **Limite Individual**: 180 minutos por ciclo
  - Cada ciclo de 3 horas deve ter media de posicao <= 180 minutos
  - Avaliacao independente por ciclo
  
- **Limite Geral**: 60 minutos acumulados por periodo
  - Soma dos 4 saldos de ciclo nao pode exceder 60 minutos
  - Mesmo que todos os ciclos sejam individuais conforme, o periodo pode ser nao conforme
  
- **Saldo**: Diferenca entre media e limite
  - Negativo = margem de seguranca (dentro do limite)
  - Positivo = excedente (fora do limite)
  - Quanto mais negativo, maior a margem de seguranca

**Validacao Implementada**:

- Contagem de linhas em cada tabela Gold
- Exibicao de resumo das agregacoes:
  - Distribuicao de status (CONFORME vs NAO CONFORME)
  - Estatisticas descritivas de `Media_Posicao_no_Ciclo` e `Saldo_Periodo`
- Verificacao de existencia de colunas calculadas
- Verificacao de tamanho dos arquivos gerados

**Formato de Saida**:

- **Parquet**: Para processamento posterior e analises
- **Excel**: Para visualizacao e compartilhamento com stakeholders
- Ambas as versoes contem os mesmos dados

## Etapa 5: Painel Consolidado

### Notebook: `05_gerar_painel_consolidado_mvp.ipynb`

#### Objetivo

Gerar painel consolidado em Excel com multiplos KMs alvo, apresentando visao executiva da conformidade por ciclo e periodo.

#### Extracao

- **Fontes**: 
  - `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/gold/gold_conformidade_ciclo_km.parquet`
  - `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/gold/gold_conformidade_periodo_km.parquet`
  - `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/dados_bronze/ciclos_concatenados.parquet` (para selecao de top KMs)

#### Transformacoes Aplicadas

**1. Selecao de Top KMs** (Reproduzindo logica de `listar_top_kms.py`):

```python
# Parametros
RODOVIA_TOP = "BR-153/GO"
SENTIDO_TOP = "S"
QTD_TOP_KM = 11

# Ler Bronze para identificar top KMs
df_bronze = pd.read_parquet(ARQ_BRONZE)

# Converter KM para numerico
km_conv = df_bronze["KM"].apply(converter_km)

# Filtrar por rodovia e sentido
mask = (df_bronze[rod_col] == RODOVIA_TOP) & (df_bronze[sent_col] == SENTIDO_TOP)
serie = km_conv[mask].dropna().round(2)

# Selecionar top N KMs por frequencia
vc = serie.value_counts().head(QTD_TOP_KM)
lista_km = [float(x) for x in list(vc.index)]
```

**Justificativa**: 
- Identificar KMs com maior volume de registros (maior representatividade estatistica)
- Garantir que o painel contenha trechos com dados suficientes para analise confiavel
- Reproduzir logica ja validada do script `listar_top_kms.py`

**2. Montagem de Blocos por Periodo**:

```python
def montar_bloco_periodo(periodo: str) -> pd.DataFrame:
    """Monta bloco de 4 linhas (ciclos) x KMs + colunas resumo."""
    if periodo == "P1":
        ciclos = [1, 2, 3, 4]
    else:
        ciclos = [5, 6, 7, 8]
    
    # Filtrar dados do periodo
    df_periodo_filtrado = ciclo_mes[
        (ciclo_mes["Periodo"] == periodo) & (ciclo_mes["Ciclo"].isin(ciclos))
    ].copy()
    
    # Criar pivot: linhas = Ciclo, colunas = KM, valores = Media_Posicao_no_Ciclo
    pivot = df_periodo_filtrado.pivot_table(
        index="Ciclo",
        columns="KM",
        values="Media_Posicao_no_Ciclo",
        aggfunc="first"
    )
    
    # Garantir que todos os ciclos e KMs estao presentes
    pivot = pivot.reindex(ciclos)
    pivot = pivot.reindex(columns=lista_km)
    
    # Renomear colunas para formato KM_X
    pivot.columns = [f"KM_{km}" for km in pivot.columns]
    
    # Calcular media entre KMs na linha do ciclo
    colunas_km = [c for c in pivot.columns if c.startswith("KM_")]
    df_bloco["Media"] = df_bloco[colunas_km].mean(axis=1)
    
    # Calcular atendimento e saldo
    df_bloco["Atendimento_180"] = np.where(
        df_bloco["Media"] <= LIMITE_INDIVIDUAL, "Sim", "Nao"
    )
    df_bloco["Saldo"] = df_bloco["Media"] - LIMITE_INDIVIDUAL
    
    return df_bloco
```

**Justificativa**: 
- Transformar dados agregados em formato tabular para visualizacao
- Pivot table organiza dados com ciclos nas linhas e KMs nas colunas
- Calcular media entre KMs para cada ciclo (visao consolidada)
- Calcular atendimento e saldo para cada ciclo

**3. Calculo de Resumos**:

```python
def montar_resumo_periodo(bloco: pd.DataFrame) -> dict:
    """Calcula resumo do periodo a partir do bloco de ciclos."""
    resumo = {"Cenario": "Cenario Concessionaria"}
    
    # Media dos ciclos (coluna Media)
    medias_validas = bloco["Media"].dropna()
    media_periodo = float(medias_validas.mean()) if len(medias_validas) > 0 else np.nan
    resumo["Media"] = media_periodo
    
    # Saldo do periodo = soma dos saldos dos 4 ciclos
    saldos_validos = bloco["Saldo"].dropna()
    saldo_periodo = float(saldos_validos.sum()) if len(saldos_validos) > 0 else np.nan
    resumo["Saldo"] = saldo_periodo
    
    # Atendimento do periodo
    resumo["Atendimento_60"] = "Sim" if not np.isnan(saldo_periodo) and saldo_periodo <= LIMITE_GERAL else "Nao"
    
    return resumo
```

**Justificativa**: 
- Calcular metricas consolidadas do periodo
- `Media`: Media das medias dos 4 ciclos (visao geral)
- `Saldo`: Soma dos saldos dos 4 ciclos (conforme regra de negocio)
- `Atendimento_60`: Avaliacao de conformidade do periodo (<= 60 min)

#### Carga

- **Destino**: `/Workspace/Users/romulobrtsilva@gmail.com/Drafts/mvp/gold/painel_consolidado_mvp_YYYYMM.xlsx`
- **Formato**: Excel com abas P1 e P2
- **Estrutura**:
  - Aba P1: 4 linhas de ciclos (1-4) + 1 linha de resumo
  - Aba P2: 4 linhas de ciclos (5-8) + 1 linha de resumo
  - Colunas: Ciclo, KM_87.26, KM_341.99, ..., Media, Atendimento_180, Saldo

#### Documentacao de Transformacao

**Estrutura do Painel**:

- **Linhas**: Ciclos (1-4 para P1, 5-8 para P2) + linha de resumo
- **Colunas**: 
  - `Ciclo`: Numero do ciclo
  - `KM_X`: Media de posicao no ciclo para cada KM alvo
  - `Media`: Media entre todos os KMs na linha do ciclo
  - `Atendimento_180`: "Sim" se Media <= 180, "Nao" caso contrario
  - `Saldo`: Media - 180 (saldo do ciclo)

**Calculos Implementados**:

1. **Media por Ciclo (colunas KM)**: Media de 30 dias da `Posicao_no_Ciclo` dos registros selecionados (1 por dia x ciclo, mais proximo do fim do ciclo)

2. **Media (coluna)**: Media entre KMs na linha do ciclo

3. **Atendimento (ciclo)**: Media <= 180 minutos

4. **Saldo (ciclo)**: Media - 180 minutos

5. **Saldo (periodo)**: Soma dos saldos dos 4 ciclos

6. **Atendimento (periodo)**: Saldo (periodo) <= 60 minutos

**Validacao Implementada**:

- Verificacao de existencia dos arquivos Gold
- Selecao automatica do mes presente nos dados
- Filtragem de dados do mes alvo e KMs selecionados
- Verificacao de criacao do arquivo Excel
- Exibicao de tamanho do arquivo gerado

## Caracteristicas do Pipeline

### Idempotencia

Todos os notebooks foram projetados para serem **idempotentes**:
- Sobrescrevem os arquivos de saida sem duplicar dados (`mode="overwrite"`)
- Podem ser executados multiplas vezes sem efeitos colaterais
- Resultados sao deterministicos (mesmos dados de entrada produzem mesmos resultados)

### Rastreabilidade

- Cada etapa preserva informacoes de origem:
  - `KM_original`: Preserva formato original do KM
  - `Marcacao`: Timestamp original preservado
  - Estrutura de diretorios clara (Bronze/Silver/Gold)
- Logs detalhados em cada notebook:
  - Contagem de registros em cada etapa
  - Estatisticas descritivas
  - Mensagens de sucesso/erro claras
- Metadados preservados:
  - Data de processamento implicita na estrutura de diretorios
  - Schema preservado em cada camada

### Escalabilidade

- **Uso de Parquet**: Formato colunar eficiente para grandes volumes
- **Filtros Previos**: Reducao de volume antes de processamentos pesados
- **Agregacoes Otimizadas**: Uso de `groupby` e `pivot_table` do Pandas
- **Selecao de Referencia**: Reducao significativa de volume (~87%)
- **Processamento Incremental**: Cada etapa processa apenas dados necessarios

### Migracao para Databricks

Os notebooks foram escritos de forma a facilitar migracao para PySpark/Delta Lake:

- **Estrutura Modular**: Cada notebook e independente e pode ser convertido para PySpark
- **Operacoes Padronizadas**: Uso de operacoes comuns (groupby, pivot, etc.) que tem equivalentes em PySpark
- **Documentacao Clara**: Transformacoes bem documentadas facilitam traducao
- **Caminhos Adaptaveis**: Uso de caminhos do workspace do Databricks
- **Tratamento de Erros**: Verificacoes de existencia de arquivos e caminhos alternativos

### Proximos Passos para Producao

1. **Conversao para PySpark**: Migrar operacoes Pandas para PySpark para processar volumes maiores
2. **Delta Lake**: Usar Delta Tables para versionamento e time travel
3. **Agendamento**: Configurar jobs no Databricks para execucao automatica
4. **Monitoramento**: Implementar alertas para falhas e metricas de qualidade
5. **Particionamento**: Particionar tabelas por mes/KM para otimizar consultas
6. **Cache**: Implementar cache de tabelas intermediarias para otimizacao

## Resumo das Transformacoes

| Etapa | Notebook | Entrada | Saida | Volume | Transformacoes Principais |
|-------|----------|---------|-------|--------|---------------------------|
| 0 | 00_gerar_bronze | Excel (2 arquivos) | Parquet Bronze | 1.548.566 | Concatenacao vertical |
| 1 | 01_analise_qualidade | Parquet Bronze | Relatorio | - | Analise de qualidade |
| 2 | 02_gerar_silver_base | Parquet Bronze | Parquet Silver Base | 1.548.566 | Padronizacao, colunas derivadas |
| 3 | 03_gerar_silver_ref | Parquet Silver Base | Parquet Silver Ref | ~197.417 | Filtros, selecao de referencia |
| 4 | 04_gerar_gold | Parquet Silver Ref | Parquet/Excel Gold | ~140.960 | Agregacoes de conformidade |
| 5 | 05_gerar_painel | Parquet Gold | Excel Painel | - | Pivot, calculos consolidados |
| 6 | 06_analise_solucao | Parquet Gold/Silver | Relatorios/Visualizacoes | - | Analise estatistica, visualizacoes |

## Conclusao

Este pipeline ETL implementa uma arquitetura de Data Lake, seguindo boas praticas de engenharia de dados. As transformacoes sao bem documentadas, idempotentes e rastreaveis, facilitando manutencao e evolucao do sistema.

