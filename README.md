# Projetofinal BI2 – Earthquake Dashboard

**Disciplina:** Business Intelligence II  
**Instituicao:** Centro Universitario de Brasilia – CEUB  
**Autores:** Davi Morais Bicalho, Matheus Perfeito Silveira

---

## Visao Geral

Este projeto tem como objetivo a construcao de um dashboard analitico em Power BI voltado para o monitoramento e analise de eventos sismicos (terremotos) registrados pelo United States Geological Survey (USGS). O trabalho engloba desde a modelagem dos dados ate a criacao de medidas DAX avancadas, passando pela construcao do pipeline de transformacao em Power Query.

A base de dados utilizada e o arquivo `earthquakes.csv`, que contem registros de terremotos com informacoes de data, localizacao geografica, magnitude, profundidade e status de revisao cientifica.

---

## Estrutura do Repositorio

```
projetofinal_bi2/
|
|-- projetofinal.SemanticModel/
|   |-- definition/
|       |-- tables/
|           |-- medidas.tmdl             # Tabela de medidas DAX
|           |-- fato_terremotos.tmdl     # Tabela fato principal
|           |-- dim_calendario.tmdl      # Dimensao de datas
|           |-- dim_classificacao.tmdl   # Dimensao de classificacao de magnitude
|           |-- dim_localizacao.tmdl     # Dimensao geografica
|           |-- dim_status_revisao.tmdl  # Dimensao de status de revisao
|
|-- README.md                            # Este documento
|-- earthquakes.csv                      # Fonte de dados bruta
```

---

## Modelagem de Dados

O modelo adota a arquitetura **Star Schema** (esquema estrela), com uma tabela fato central relacionada a quatro tabelas de dimensao. Essa escolha garante desempenho otimizado nas consultas DAX, reducao de redundancia e facilidade de manutencao.

### Tabela Fato: `fato_terremotos`

Tabela central do modelo. Armazena os indicadores quantitativos de cada evento sismico, com chaves estrangeiras para todas as dimensoes.

| Coluna        | Tipo    | Descricao                                          |
|---------------|---------|----------------------------------------------------|
| `usgs_id`     | Texto   | Identificador unico do evento, atribuido pelo USGS |
| `depth_km`    | Decimal | Profundidade hipocentral do terremoto em quilometros |
| `magnitude`   | Decimal | Magnitude do terremoto na escala registrada        |
| `id_status`   | Inteiro | Chave estrangeira para `dim_status_revisao`        |
| `id_mag`      | Inteiro | Chave estrangeira para `dim_classificacao`         |
| `id_loc`      | Inteiro | Chave estrangeira para `dim_localizacao`           |
| `id_calendario`| Inteiro| Chave estrangeira para `dim_calendario`            |

### Dimensao: `dim_calendario`

Gerada a partir das datas unicas presentes no dataset. Contem as colunas `date` (tipo Date), `year`, `month` e `id_calendario` (chave surrogada incremental com passo 4).

### Dimensao: `dim_classificacao`

Armazena as combinacoes unicas de tipo de magnitude (`mag_type`) e classe de magnitude (`magnitude_class`). Chave surrogada `id_mag` com incremento de 3.

### Dimensao: `dim_localizacao`

Armazena os registros unicos de localizacao geografica, contendo `lat` (Latitude), `lon` (Longitude) e `location_desc`. Inclui uma coluna calculada (`hemisferio`) descrita na secao de colunas calculadas. Chave surrogada `id_loc` com incremento de 1.

### Dimensao: `dim_status_revisao`

Armazena os valores distintos de `review_status` (ex.: `reviewed`, `automatic`). Chave surrogada `id_status` com incremento de 1.

---

## Pipeline de Transformacao (Power Query / M)

Todas as tabelas sao carregadas a partir do mesmo arquivo-fonte `earthquakes.csv` (UTF-8, 12 colunas, delimitador virgula). Cada tabela passa por etapas especificas de limpeza e preparacao.

### fato_terremotos

1. **Leitura do CSV** com tipos inicialmente inferidos.
2. **Promocao de cabecalhos.**
3. **Conversao de tipos** com localidade `en-US` para `lat`, `lon`, `depth_km` e `magnitude` (garante ponto decimal correto independente da configuracao regional).
4. **Conversao de `date`** para tipo `Date`.
5. **Mesclagem com `dim_status_revisao`** via `review_status` (Left Outer Join) para obtencao de `id_status`.
6. **Mesclagem com `dim_classificacao`** via `mag_type` e `magnitude_class` para obtencao de `id_mag`.
7. **Mesclagem com `dim_localizacao`** via `lat`, `lon` e `location_desc` para obtencao de `id_loc`.
8. **Mesclagem com `dim_calendario`** via `date` para obtencao de `id_calendario`.
9. **Remocao das colunas originais** substituidas pelas chaves surrogadas (`date`, `year`, `month`, `lat`, `lon`, `mag_type`, `magnitude_class`, `location_desc`, `review_status`).
10. **Conversao final de `magnitude`** para tipo numerico.
11. **Ordenacao descendente** por `magnitude` e `depth_km`.

### dim_calendario

1. Leitura, promocao de cabecalhos e conversao de tipos.
2. Remocao das colunas irrelevantes, mantendo apenas `date`, `year` e `month`.
3. **Remocao de duplicatas** pela coluna `date`.
4. **Adicao de coluna de indice** (`id_calendario`), iniciando em 1 com passo 4.
5. Renomeacao do indice para `id_calendario`.

### dim_classificacao

1. Leitura, promocao de cabecalhos e conversao de tipos.
2. Remocao de todas as colunas exceto `mag_type` e `magnitude_class`.
3. **Adicao de coluna de indice** (`id_mag`), iniciando em 1 com passo 3.
4. Renomeacao e **remocao de duplicatas** pela combinacao `magnitude_class` + `mag_type`.

### dim_localizacao

1. Leitura, promocao de cabecalhos e conversao de tipos com localidade `en-US` para `lat` e `lon`.
2. Remocao das colunas irrelevantes, mantendo `lat`, `lon`, `location_desc` e `magnitude`.
3. Remocao da coluna `magnitude`.
4. **Remocao de duplicatas** pelo conjunto `location_desc`, `lon`, `lat`.
5. **Adicao de coluna de indice** (`id_loc`) com passo 1.

### dim_status_revisao

1. Leitura, promocao de cabecalhos e conversao de tipos.
2. Selecao exclusiva da coluna `review_status`.
3. **Remocao de duplicatas**.
4. **Adicao de coluna de indice** (`id_status`) com passo 1.
5. Aplicacao de filtro de manutencao (all rows = true).

---

## Colunas Calculadas DAX

### `dim_localizacao[hemisferio]`

Classifica cada localizacao no hemisferio Norte ou Sul com base na latitude.

```dax
hemisferio = IF(dim_localizacao[lat] < 0, "Sul", "Norte")
```

**Funcoes utilizadas:** `IF`

---

## Medidas DAX

Todas as medidas estao centralizadas na tabela `medidas`, que e uma tabela vazia utilizada apenas como container logico — pratica recomendada para organizacao em modelos Power BI.

### Medidas de Contagem e Volumetria

**`qtd_total_terremotos`**
Conta o total de eventos sismicos registrados no contexto de filtro ativo.
```dax
qtd_total_terremotos = COUNT(fato_terremotos[usgs_id])
```
Formato: `0`  
Funcoes: `COUNT`

---

**`qtd_locais_unicos`**
Conta o numero de localizacoes distintas presentes no contexto filtrado.
```dax
qtd_locais_unicos = DISTINCTCOUNT(dim_localizacao[location_desc])
```
Formato: `0`  
Funcoes: `DISTINCTCOUNT`

---

**`qtd_terremotos_ano`**
Acumula a quantidade de terremotos do inicio do ano ate a data corrente no contexto.
```dax
qtd_terremotos_ano = TOTALYTD([qtd_total_terremotos], dim_calendario[date])
```
Formato: `0`  
Funcoes: `TOTALYTD`

---

**`qtd_terremotos_mes`**
Acumula a quantidade de terremotos do inicio do mes ate a data corrente no contexto.
```dax
qtd_terremotos_mes = TOTALMTD([qtd_total_terremotos], dim_calendario[date])
```
Formato: `0`  
Funcoes: `TOTALMTD`

---

**`qtd_terremotos_revisados`**
Filtra e conta apenas os terremotos cujo status de revisao e `"reviewed"`.
```dax
qtd_terremotos_revisados =
    CALCULATE(
        [qtd_total_terremotos],
        dim_status_revisao[review_status] = "reviewed"
    )
```
Formato: `0`  
Funcoes: `CALCULATE`

---

**`qtd_terremotos_acumulado_geral`**
Calcula o total acumulado (running total) de terremotos desde o inicio do historico ate a data maxima presente no contexto. Ignora o filtro da dimensao de calendario para percorrer todo o intervalo.
```dax
qtd_terremotos_acumulado_geral =
    CALCULATE(
        [qtd_total_terremotos],
        FILTER(
            ALL(dim_calendario),
            dim_calendario[date] <= MAX(dim_calendario[date])
        )
    )
```
Formato: `0`  
Funcoes: `CALCULATE`, `FILTER`, `ALL`, `MAX`

---

### Medidas Temporais e de Comparacao

**`qtd_terremotos_ano_anterior`**
Retorna a quantidade de terremotos no mesmo periodo do ano anterior, utilizando deslocamento de -1 ano na dimensao de calendario.
```dax
qtd_terremotos_ano_anterior =
    CALCULATE(
        [qtd_total_terremotos],
        DATEADD(dim_calendario[date], -1, YEAR)
    )
```
Formato: `0`  
Funcoes: `CALCULATE`, `DATEADD`

---

**`crescimento_yoy_terremotos`**
Calcula a variacao percentual ano a ano (Year-over-Year) da quantidade de terremotos. Retorna BLANK quando nao ha periodo anterior disponivel para comparacao.
```dax
crescimento_yoy_terremotos =
    VAR _atual    = [qtd_total_terremotos]
    VAR _anterior = [qtd_terremotos_ano_anterior]
    RETURN
        IF(
            NOT ISBLANK(_anterior),
            DIVIDE(_atual - _anterior, _anterior)
        )
```
Formato: `0.00%`  
Funcoes: `VAR`, `RETURN`, `IF`, `NOT`, `ISBLANK`, `DIVIDE`

---

### Medidas Estatisticas

**`magnitude_media`**
Calcula a media aritmetica da magnitude dos terremotos no contexto filtrado.
```dax
magnitude_media = AVERAGE(fato_terremotos[magnitude])
```
Formato: `0.00`  
Funcoes: `AVERAGE`

---

**`profundidade_media`**
Calcula a media aritmetica da profundidade hipocentral em quilometros.
```dax
profundidade_media = AVERAGE(fato_terremotos[depth_km])
```
Formato: `0.00`  
Funcoes: `AVERAGE`

---

**`magnitude_maxima`**
Retorna o maior valor de magnitude registrado no contexto filtrado.
```dax
magnitude_maxima = MAX(fato_terremotos[magnitude])
```
Formato: `0.00`  
Funcoes: `MAX`

---

**`profundidade_maxima`**
Retorna o maior valor de profundidade registrado no contexto filtrado.
```dax
profundidade_maxima = MAX(fato_terremotos[depth_km])
```
Formato: `0.00`  
Funcoes: `MAX`

---

**`magnitude_media_global`**
Calcula a media de magnitude de todos os terremotos do dataset, ignorando qualquer filtro de contexto aplicado sobre a tabela fato. Util para comparacao de referencia em cards e graficos.
```dax
magnitude_media_global =
    CALCULATE(
        [magnitude_media],
        ALL(fato_terremotos)
    )
```
Formato: `0.00`  
Funcoes: `CALCULATE`, `ALL`

---

### Medidas de Qualidade dos Dados

**`percentual_terremotos_revisados`**
Calcula a proporcao de terremotos revisados em relacao ao total do dataset, removendo o filtro de status para obter o denominador correto.
```dax
percentual_terremotos_revisados =
    DIVIDE(
        [qtd_terremotos_revisados],
        CALCULATE([qtd_total_terremotos], ALL(dim_status_revisao))
    )
```
Formato: `0.00%`  
Funcoes: `DIVIDE`, `CALCULATE`, `ALL`

---

## Resumo das Funcoes DAX Utilizadas

| Funcao          | Categoria            | Aplicacao no Projeto                                      |
|-----------------|----------------------|-----------------------------------------------------------|
| `COUNT`         | Agregacao            | Contagem de eventos por `usgs_id`                         |
| `AVERAGE`       | Agregacao            | Media de magnitude e profundidade                         |
| `MAX`           | Agregacao            | Maximos de magnitude e profundidade                       |
| `DISTINCTCOUNT` | Agregacao            | Contagem de localizacoes unicas                           |
| `CALCULATE`     | Modificacao de contexto | Base de todas as medidas com filtro dinamico           |
| `ALL`           | Modificacao de contexto | Remove filtros para calculos globais                   |
| `FILTER`        | Tabela               | Filtragem condicional para running total                  |
| `TOTALYTD`      | Time Intelligence    | Acumulado anual                                           |
| `TOTALMTD`      | Time Intelligence    | Acumulado mensal                                          |
| `DATEADD`       | Time Intelligence    | Deslocamento temporal para comparacao YoY                 |
| `SAMEPERIODLASTYEAR` | Time Intelligence | Periodo equivalente no ano anterior                  |
| `DIVIDE`        | Matematica           | Divisao segura com tratamento de denominador zero         |
| `IF`            | Logica               | Condicional para evitar resultados em branco              |
| `ISBLANK`       | Informacao           | Verificacao de valores em branco                          |
| `VAR / RETURN`  | Variaveis            | Encapsulamento de calculos intermediarios                 |

---

## Como Executar o Projeto

1. Instale o **Power BI Desktop** (versao mais recente).
2. Abra o arquivo do modelo contido na pasta `projetofinal.SemanticModel`.
3. Na primeira abertura, confirme o caminho do arquivo `earthquakes.csv` nas configuracoes de fonte de dados.
4. Atualize os dados. O Power Query executara todo o pipeline de transformacao automaticamente.
5. As medidas DAX estarao disponiveis para uso nos visuais de todas as paginas do relatorio.

---

## Dependencias

- Power BI Desktop
- Arquivo `earthquakes.csv` (incluido no repositorio)

---

## Autores

Davi Morais Bicalho  
Matheus Perfeito Silveira
