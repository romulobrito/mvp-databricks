# Catalogo de Dados - MVP Analise de Conformidade de Ciclos

## Linhagem dos Dados (Data Lineage)

### Origem dos Dados

**Fonte**: Agencia Nacional de Transportes Terrestres (ANTT)

**Formato Original**: Arquivos Excel contendo registros de GPS de viaturas de patrulhamento rodoviario

**Arquivos Originais**:
- `Ciclo01.xlsx`: 752.223 registros
- `Ciclo02.xlsx`: 796.343 registros
- **Total Bruto**: 1.548.566 registros

**Periodo dos Dados**: Maio de 2025 (01/05/2025 a 31/05/2025)

**Localizacao**: Os arquivos Excel estao localizados no diretorio raiz do projeto, fora da pasta `mvp/`

### Tecnica de Composicao

**Metodo**: Concatenacao vertical de multiplos arquivos Excel

**Implementacao**: Funcao `carregar_ciclos_concatenados()` do modulo `carregar_ciclos_concatenados.py`

**Processo**:
1. Leitura dos arquivos Excel usando `pandas.read_excel()`
2. Concatenacao vertical dos DataFrames com `pd.concat(ignore_index=True)`
3. Preservacao de todas as colunas originais sem modificacao
4. Persistencia em formato Parquet na camada Bronze

**Justificativa**: A concatenacao permite unificar dados de multiplos arquivos em uma unica tabela para processamento posterior, mantendo a integridade dos dados originais.

### Fluxo de Transformacao (Linhagem Completa)

```
Arquivos Excel Originais (Ciclo01.xlsx, Ciclo02.xlsx)
    |
    | [Concatenacao Vertical]
    |
Bronze: ciclos_concatenados.parquet (1.548.566 registros)
    |
    | [Tratamento: conversao tipos, colunas derivadas]
    |
Silver Base: silver_ciclos_base.parquet (1.548.566 registros)
    |
    | [Filtros + Selecao de referencia]
    |
Silver Referencia: silver_ciclos_referencia.parquet (~197.417 registros)
    |
    | [Agregacao por Mes x KM x Ciclo]
    |
Gold: gold_conformidade_ciclo_km.parquet (agregacoes)
    |
    | [Agregacao por Mes x KM x Periodo]
    |
Gold: gold_conformidade_periodo_km.parquet (agregacoes)
```

### Licenca de Uso

Os dados sao de uso interno da ANTT e foram fornecidos para fins de analise academica. E necessario cuidado com a confidencialidade dos dados e das analises realizadas.

## Camada Bronze: `ciclos_concatenados`

### Descricao

Dados brutos concatenados dos arquivos Excel originais, sem tratamento ou transformacao.

### Tabela: `bronze.ciclos_concatenados`

| Coluna | Tipo | Descricao | Dominio | Valores Min/Max ou Categorias |
|--------|------|-----------|---------|-------------------------------|
| `DataEv` | string | Data do evento (formato YYYY-MM-DD) | Data valida | **Min**: 2025-05-01<br>**Max**: 2025-05-31 |
| `HoraEv` | string | Hora do evento (formato HH:MM:SS) | Hora valida | **Min**: 00:00:00<br>**Max**: 23:59:59 |
| `CodRecurso` | integer | Codigo numerico do recurso/viatura | Codigo positivo | **Min**: 131<br>**Max**: 999<br>**Valores observados**: 131, 132, 133, ..., 999 |
| `Recursos` | string | Nome/identificacao do recurso/viatura | Categorico | **Categorias observadas**:<br>- GP01, GP02, GP03, GP04 (RESERVA)<br>- GL02, GL03, GL04, GL05, GL08, GL09<br>- IT04, IT07, IT09, IT10, IT14 (RESERVA)<br>- CB01, CP03<br>- R12, R16, R19<br>- A02<br>**Total**: ~61 viaturas distintas |
| `Latitude` | float | Coordenada de latitude GPS (graus decimais) | Latitude valida | **Min**: -15.826255<br>**Max**: -11.765139<br>**Media**: ~-14.31 |
| `Longitude` | float | Coordenada de longitude GPS (graus decimais) | Longitude valida | **Min**: -49.281891<br>**Max**: -49.100357<br>**Media**: ~-49.11 |
| `SiglaRodovia` | string | Sigla da rodovia | Categorico | **Categorias**:<br>- "BR-153/GO"<br>- "BR-153/TO"<br>- "BR-414"<br>- "BR-080" |
| `Sentido` | string | Sentido da rodovia | Categorico | **Categorias**:<br>- "N" (Norte)<br>- "S" (Sul) |
| `KM` | string | Quilometragem no formato XXX+YYY | String formatada | **Formato**: "XXX+YYY" onde XXX e YYY sao numeros<br>**Exemplos**: "212+768", "367+530", "677+437"<br>**Valores observados**: Variam de "1+001" a "998+000" |

### Estatisticas Basicas

- **Total de registros**: 1.548.566
- **Periodo coberto**: 2025-05-01 a 2025-05-31 (31 dias)
- **Rodovias presentes**: BR-153/GO, BR-153/TO, BR-414, BR-080
- **Sentidos**: Norte (N) e Sul (S)

## Camada Silver: `silver_ciclos_base`

### Descricao

Dados tratados com conversao de tipos, criacao de colunas derivadas e padronizacao de nomes.

### Transformacoes Aplicadas

1. **Renomeacao de colunas**:
   - `DataEv` -> `DataOcorrencia`
   - `HoraEv` -> `HoraAcionamento`
   - `CodRecurso` -> `CodigoRecurso`
   - `Recursos` -> `Recurso`
   - `SiglaRodovia` -> `Rodovia`

2. **Criacao de `Marcacao`**: Timestamp combinando `DataOcorrencia` + `HoraAcionamento`

3. **Conversao de `KM`**: De formato string "XXX+YYY" para float XXX.YYY

4. **Colunas derivadas**:
   - `Data`: Data extraida de `Marcacao`
   - `Ciclo`: Numero do ciclo (1-8) calculado como `hora // 3 + 1`
   - `Minutos`: Minutos desde 00:00 calculado como `hora*60 + minuto + segundo/60`
   - `Inicio_Ideal`: Inicio ideal do ciclo em minutos `(Ciclo-1) * 180`
   - `Fim_Ideal`: Fim ideal do ciclo em minutos `Ciclo * 180`

### Tabela: `silver.ciclos_base`

| Coluna | Tipo | Descricao | Dominio | Valores Min/Max ou Categorias |
|--------|------|-----------|---------|-------------------------------|
| `Marcacao` | timestamp | Timestamp completo do evento | Timestamp valido | **Min**: 2025-05-01 00:00:05<br>**Max**: 2025-05-31 23:59:56 |
| `Data` | date | Data do evento | Data valida | **Min**: 2025-05-01<br>**Max**: 2025-05-31 |
| `Rodovia` | string | Sigla da rodovia | Categorico | **Categorias**:<br>- "BR-153/GO"<br>- "BR-153/TO"<br>- "BR-414"<br>- "BR-080" |
| `Sentido` | string | Sentido da rodovia | Categorico | **Categorias**:<br>- "N" (Norte)<br>- "S" (Sul) |
| `KM` | float | Quilometragem convertida (XXX.YYY) | Numerico positivo | **Min**: 1.001<br>**Max**: 998.0<br>**Media**: ~357.99<br>**Desvio Padrao**: ~223.75 |
| `KM_original` | string | KM original (formato XXX+YYY) | String formatada | Preservado para auditoria<br>**Formato**: "XXX+YYY" |
| `Recurso` | string | Nome do recurso/viatura | Categorico | **Mesmas categorias da camada Bronze**<br>**Total**: ~61 viaturas distintas |
| `CodigoRecurso` | integer | Codigo numerico do recurso | Codigo positivo | **Min**: 131<br>**Max**: 999 |
| `Ciclo` | integer | Numero do ciclo (1-8) | Categorico ordenado | **Categorias**:<br>- 1 (00:00-02:59)<br>- 2 (03:00-05:59)<br>- 3 (06:00-08:59)<br>- 4 (09:00-11:59)<br>- 5 (12:00-14:59)<br>- 6 (15:00-17:59)<br>- 7 (18:00-20:59)<br>- 8 (21:00-23:59) |
| `Minutos` | float | Minutos desde 00:00 | Numerico | **Min**: 0.0<br>**Max**: 1439.983<br>**Teorico Max**: 1440.0 (24 horas) |
| `Inicio_Ideal` | float | Inicio ideal do ciclo (minutos) | Numerico fixo | **Valores possiveis**:<br>- 0 (Ciclo 1)<br>- 180 (Ciclo 2)<br>- 360 (Ciclo 3)<br>- 540 (Ciclo 4)<br>- 720 (Ciclo 5)<br>- 900 (Ciclo 6)<br>- 1080 (Ciclo 7)<br>- 1260 (Ciclo 8) |
| `Fim_Ideal` | float | Fim ideal do ciclo (minutos) | Numerico fixo | **Valores possiveis**:<br>- 180 (Ciclo 1)<br>- 360 (Ciclo 2)<br>- 540 (Ciclo 3)<br>- 720 (Ciclo 4)<br>- 900 (Ciclo 5)<br>- 1080 (Ciclo 6)<br>- 1260 (Ciclo 7)<br>- 1440 (Ciclo 8) |

### Filtros Aplicados

- Registros com `Marcacao` invalida sao removidos (0 registros removidos no dataset atual)

## Camada Silver: `silver_ciclos_referencia`

### Descricao

Selecao de 1 registro por combinacao de `Data x Ciclo x Rodovia x Sentido x KM x Recurso`, escolhendo o registro mais proximo do fim ideal do ciclo (preferindo antes do limite).

### Transformacoes Aplicadas

1. **Filtros previos**:
   - Rodovia: "BR-153/GO" (configuravel)
   - Sentido: "S" (configuravel)
   - Tolerancia de tempo: ±15 minutos em relacao ao ciclo

2. **Logica de selecao**:
   - `Preferencia`: 0 se `Minutos <= Fim_Ideal`, 1 caso contrario
   - `Delta_limite`: Distancia ate o `Fim_Ideal` dentro de cada preferencia
   - Ordenacao: `Data, Ciclo, Rodovia, Sentido, KM, Recurso, Preferencia, Delta_limite, Marcacao`
   - Selecao: Primeiro registro de cada grupo (`.groupby().first()`)

3. **Coluna derivada**:
   - `Minuto_no_Ciclo`: Posicao dentro do ciclo (0-180 min) = `Minutos - Inicio_Ideal`

### Tabela: `silver.ciclos_referencia`

| Coluna | Tipo | Descricao | Dominio | Valores Min/Max ou Categorias |
|--------|------|-----------|---------|-------------------------------|
| `Data` | date | Data do evento | Data valida | **Min**: 2025-05-01<br>**Max**: 2025-05-31 |
| `Ciclo` | integer | Numero do ciclo | Categorico ordenado | **Categorias**: 1, 2, 3, 4, 5, 6, 7, 8 |
| `Rodovia` | string | Sigla da rodovia | Categorico fixo | **Categoria**: "BR-153/GO" (apos filtro) |
| `Sentido` | string | Sentido da rodovia | Categorico fixo | **Categoria**: "S" (apos filtro) |
| `KM` | float | Quilometragem | Numerico positivo | **Min observado**: ~1.0<br>**Max observado**: ~998.0<br>**Valores**: Variam conforme trechos monitorados |
| `Recurso` | string | Nome do recurso/viatura | Categorico | **Categorias**: Subset das ~61 viaturas presentes em BR-153/GO/S<br>**Principais**: GL02, GL03, IT07, GP02, GL04, CB01, GL09, GP01, IT14, IT04 |
| `Marcacao` | timestamp | Timestamp do registro selecionado | Timestamp valido | **Min**: 2025-05-01 00:00:05<br>**Max**: 2025-05-31 23:59:56 |
| `Minuto_no_Ciclo` | float | Posicao dentro do ciclo (minutos) | Numerico | **Min**: 0.0<br>**Max**: 179.983<br>**Media**: ~98.37<br>**Mediana**: ~102.03<br>**Desvio Padrao**: ~52.93<br>**Dominio teorico**: [0, 180) |

### Estatisticas

- **Total de registros**: ~197.417 (apos filtros e selecao)
- **Reducao**: De 1.548.566 para 197.417 (selecao de 1 registro por chave)

## Camada Gold: `gold_conformidade_ciclo_km`

### Descricao

Agregacao por `Mes x KM x Ciclo x Periodo` com calculo de medias, saldos e status de conformidade por ciclo.

### Transformacoes Aplicadas

1. **Agregacao**:
   - Agrupa por `Mes, KM, Ciclo, Periodo`
   - Calcula `Media_Posicao_no_Ciclo` = media de `Minuto_no_Ciclo`

2. **Calculos de conformidade**:
   - `Saldo_Ciclo` = `Media_Posicao_no_Ciclo - 180`
   - `Status_Ciclo` = "CONFORME" se `Media_Posicao_no_Ciclo <= 180`, senao "NAO CONFORME"

3. **Definicao de Periodo**:
   - P1: Ciclos 1-4 (00:00-11:59)
   - P2: Ciclos 5-8 (12:00-23:59)

### Tabela: `gold.conformidade_ciclo_km`

| Coluna | Tipo | Descricao | Dominio | Valores Min/Max ou Categorias |
|--------|------|-----------|---------|-------------------------------|
| `Mes` | string | Mes no formato AAAA-MM | Categorico | **Categoria**: "2025-05" |
| `KM` | float | Quilometragem | Numerico positivo | **Min observado**: ~1.0<br>**Max observado**: ~998.0<br>**Valores**: Variam conforme trechos analisados |
| `Ciclo` | integer | Numero do ciclo | Categorico ordenado | **Categorias**: 1, 2, 3, 4, 5, 6, 7, 8 |
| `Periodo` | string | Periodo (P1 ou P2) | Categorico | **Categorias**:<br>- "P1" (Ciclos 1-4: 00:00-11:59)<br>- "P2" (Ciclos 5-8: 12:00-23:59) |
| `Media_Posicao_no_Ciclo` | float | Media mensal da posicao no ciclo (minutos) | Numerico | **Min teorico**: 0.0<br>**Max teorico**: 180.0<br>**Min observado**: ~0.0<br>**Max observado**: ~179.98<br>**Media geral**: ~91.0<br>**Interpretacao**: Media de 30 dias da `Minuto_no_Ciclo` |
| `Saldo_Ciclo` | float | Saldo do ciclo (Media - 180) | Numerico | **Min observado**: ~-180.0<br>**Max observado**: ~-0.02<br>**Media geral**: ~-89.0<br>**Interpretacao**:<br>- Valores negativos: margem de seguranca<br>- Valores positivos: excedente (nao conforme) |
| `Status_Ciclo` | string | Status de conformidade | Categorico | **Categorias**:<br>- "CONFORME" (se Media <= 180)<br>- "NAO CONFORME" (se Media > 180) |

### Limites de Conformidade

- **Limite Individual**: 180 minutos por ciclo
- **Limite Geral**: 60 minutos acumulados por periodo

## Camada Gold: `gold_conformidade_periodo_km`

### Descricao

Agregacao por `Mes x KM x Periodo` com calculo de saldo acumulado e status de conformidade por periodo.

### Transformacoes Aplicadas

1. **Agregacao**:
   - Agrupa por `Mes, KM, Periodo`
   - Calcula `Saldo_Periodo` = soma dos 4 `Saldo_Ciclo` do periodo

2. **Calculos de conformidade**:
   - `Status_Periodo` = "CONFORME" se `Saldo_Periodo <= 60`, senao "NAO CONFORME"

### Tabela: `gold.conformidade_periodo_km`

| Coluna | Tipo | Descricao | Dominio | Valores Min/Max ou Categorias |
|--------|------|-----------|---------|-------------------------------|
| `Mes` | string | Mes no formato AAAA-MM | Categorico | **Categoria**: "2025-05" |
| `KM` | float | Quilometragem | Numerico positivo | **Min observado**: ~1.0<br>**Max observado**: ~998.0<br>**Valores**: Variam conforme trechos analisados |
| `Periodo` | string | Periodo (P1 ou P2) | Categorico | **Categorias**:<br>- "P1" (Ciclos 1-4: 00:00-11:59)<br>- "P2" (Ciclos 5-8: 12:00-23:59) |
| `Saldo_Periodo` | float | Saldo acumulado do periodo (minutos) | Numerico | **Calculo**: Soma dos 4 `Saldo_Ciclo` do periodo<br>**Min teorico**: -720.0 (4 ciclos * -180)<br>**Max teorico**: +infinito<br>**Min observado**: ~-400.0<br>**Max observado**: ~-0.02<br>**Media geral**: ~-105.0<br>**Interpretacao**:<br>- Valores <= 60: conforme<br>- Valores > 60: nao conforme |
| `Status_Periodo` | string | Status de conformidade | Categorico | **Categorias**:<br>- "CONFORME" (se Saldo <= 60)<br>- "NAO CONFORME" (se Saldo > 60) |

## Observacoes Importantes

1. **Pre-processamento**: Os dados brutos passam por uma etapa de selecao de registros de referencia, reduzindo significativamente o volume antes das agregacoes Gold (de ~1.5M para ~197K registros).

2. **Filtros Configuraveis**: Os filtros de rodovia, sentido e tolerancia de tempo podem ser ajustados no notebook `02_gerar_silver_ciclos_referencia.ipynb`. Os valores padrao sao:
   - Rodovia: "BR-153/GO"
   - Sentido: "S"
   - Tolerancia: ±15 minutos

3. **Granularidade**: A camada Gold mantem granularidade por KM, permitindo analises espaciais detalhadas. Cada registro representa uma agregacao mensal por combinacao de Mes x KM x Ciclo (ou Periodo).

4. **Idempotencia**: Os notebooks foram projetados para serem executados multiplas vezes sem duplicar dados (sobrescrevem os arquivos de saida).

5. **Valores Observados vs Teoricos**: 
   - Valores observados sao baseados no dataset atual (maio/2025)
   - Valores teoricos representam limites contratuais ou limites fisicos/logicos
   - Alguns valores podem variar conforme novos dados sejam adicionados

6. **Campos Categoricos**: 
   - Campos categoricos com poucos valores possuem lista completa de categorias
   - Campos categoricos com muitos valores (ex: `Recurso`) possuem lista parcial com principais categorias
   - O total de categorias unicas pode ser obtido consultando os dados diretamente

