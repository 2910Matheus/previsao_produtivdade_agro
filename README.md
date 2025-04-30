# Projeto: Previsão de Produtividade Agrícola na Região de Palotina, PR

Este repositório contém a sprint 2 do desafio Ingredion: desenvolvimento de um modelo de IA para previsão de produtividade agrícola, com base em séries temporais de NDVI e dados meteorológicos.

Foi utilizado como base de dados do https://portal.inmet.gov.br/ e www.satveg.cnptia.embrapa.br

---

## 📂 Estrutura do Repositório

```
├── data
│   ├── dados_mapas_inmet_palotina_2015 a 2025.csv          # Dados inmet
│   └── satveg_planilha.xlsx                                # Dados satveg
│
│
├── scripts                # Notebook
│   ├── ModeloIA.ipynb     # Modelo IA com o carregamento e a segmentação
│
├── bestModel              # Modelos treinados e objetos salvos
│   ├── melhor_modelo.pkl  # Modelo selecionado
│   └── scaler.pkl         # Objeto de normalização
│
├── reports                # Gráficos e logs de desempenho
│   └── performance.png    # Comparação real vs predito
│
└── README.md              # Documentação geral (este arquivo)
```

---

## 1. Pré-processamento dos Dados

### 1.1. Carregamento e Tratamento

- **NDVI**: planilha Excel (`satveg_planilha.xlsx`) carregada com `pd.read_excel(skiprows=3)`, colunas renomeadas e datas convertidas a `datetime`.
- **Clima**: arquivos CSV anuais (`dados_mapas_inmet_palotina_{ano}.csv`), limpeza de cabeçalhos, conversão numérica e indexação por data.
- **Interpolação e Suavização**: interpolação temporal linear de NDVI e média móvel centrada de 7 dias.
- - A interpolação remove lacunas e ruídos ocasionais.
- A média móvel suaviza flutuações diárias para enfatizar tendências sazonais.


### 1.2. Justificativa das variáveis

Utilizamos o índice NDVI como principal indicador da saúde da vegetação, pois ele reflete diretamente a produtividade agrícola. 
Além disso, incorporamos dados meteorológicos (precipitação, temperatura máxima e mínima) por sua influência direta nas fases críticas de crescimento das culturas.


---

## 2. Análise Exploratória (EDA)


Durante a análise exploratória, observamos correlações sazonais no NDVI com os períodos de crescimento agrícola.
Abaixo, gráficos representando essa variação ao longo do tempo foram gerados no notebook `ModeloIA.ipynb` (ver pasta `reports/`). 
A sazonalidade reflete picos de vegetação durante os meses chuvosos, coincidindo com maior produtividade.


No notebook `script/ModeloIA.ipynb`, realizamos as seguintes análises:

1. **Série Temporal de NDVI (2000–2025)**
   - Média anual de NDVI: **0.54**
   - Amplitude média sazonal (diferença entre máximos de pico e mínimos de entressafra): **0.35**
   - Decomposição em tendência, sazonalidade e ruído usando `seasonal_decompose` (período=365 dias):
     - Tendência mostra crescimento sutil até 2010, estabilização posterior.
     - Sazonalidade evidencia picos regulares de novembro a janeiro.

2. **Correlação Entre Variáveis**
   - Coeficiente de correlação de Pearson (ano agrícola):
     - NDVI_sum × Produtividade: **0.62**
     - Precipitação × Produtividade: **0.48**
     - Temperatura Média × Produtividade: **–0.15**
   - Matrizes de correlação e scatter plots estão no notebook.

3. **Autocorrelação de Produtividade**
   - Função de autocorrelação (ACF) até lag 5:
     - Lag-1: **0.55**, Lag-2: **0.34**, Lag-3: **0.20**.
   - Sugere dependência forte de ano anterior.

4. **Distribuição das Variáveis**
   - Histogramas de NDVI_max, NDVI_mean, Precipitação e Produtividade.
   - Outliers identificados em produtividade para anos de seca (2005, 2012).

5. **Segmentação e Máscara de Cultivo**
   - Aplicamos threshold NDVI > 0.4 e K-Means (k=3) para isolar regiões vegetadas.
   - Mascara resultante cobre em média **42%** da área total de análise.
   - Visualização da máscara sobreposta em imagem GeoTIFF no notebook.

## 3. Segmentação de Áreas de Cultivo

Demonstrado em `script/ModeloIA.ipynb` com:

- **Thresholding** de NDVI (e.g. NDVI > 0.4) para mascarar vegetação ativa.
- **Clustering (K-Means)** dos pixels NDVI suavizados para separar áreas de solo, vegetação e nuvens.
- Máscara aplicada a uma matriz de pixels extraída de imagens GeoTIFF.

```python
# Exemplo de thresholding simples
mask = ndvi_array > 0.4  # True nas áreas de plantações
segmented = np.where(mask, ndvi_array, np.nan)
```

Esta segmentação permite focar a extração de features apenas nos pixels representativos de lavoura.

---

## 4. Extração de Features

Implementado em `script/ModeloIA.ipynb`:

- Estatísticas por Ano Agrícola (Jul–Jun): `max, min, mean, sum` de NDVI suavizado.
- Variáveis derivadas:
  - `NDVI_mean_3y`: média móvel de 3 anos.
  - `Taxa_Crescimento_NDVI`: variação percentual ano a ano.
  - `Estresse_Hídrico`: razão precipitação/temperatura média.
  - `Produtividade_lag1`: produtividade defasada em 1 ano.

**Justificativas**:

- `NDVI_mean_3y` captura tendência de longo prazo.
- `Taxa_Crescimento_NDVI` sinaliza variações abruptas.
- `Estresse_Hídrico` correlaciona déficit hídrico e impacto na safra.

---

## 5. Construção e Validação do Modelo

Para prever a produtividade agrícola com base em séries temporais multivariadas (NDVI e clima), optamos por modelos de aprendizado supervisionado robustos frente a dados não-lineares e com variáveis correlacionadas.

### 🔍 Modelos Avaliados

Testamos dois algoritmos de regressão amplamente utilizados em problemas de séries temporais com dependência entre variáveis:

- **Random Forest Regressor**: útil para capturar relações não lineares e interações entre variáveis, com baixa sensibilidade a outliers.
  - Hiperparâmetros testados: `max_depth = [3, 5]`, `n_estimators = [100, 200]`

- **XGBoost Regressor**: modelo baseado em árvores com boosting, eficaz para capturar padrões complexos e diferenças sazonais.
  - Hiperparâmetros testados: `max_depth = [2, 3]`, `learning_rate = [0.1, 0.05]`

### ⏳ Estratégia de Validação

Utilizamos a validação temporal com `TimeSeriesSplit(n_splits=3, gap=2)` para evitar vazamento de informação entre anos consecutivos, garantindo que o modelo fosse avaliado apenas com dados futuros em relação ao treinamento.

### 📏 Métrica de Seleção

A principal métrica adotada foi o **erro quadrático médio negativo** (`neg_mean_squared_error`), por sua sensibilidade a grandes desvios, refletindo com precisão falhas na previsão de produtividade.

### 🏆 Modelo Selecionado

O modelo com melhor desempenho foi o **XGBoost Regressor** com:
- `max_depth = 2`
- `learning_rate = 0.1`

### Justificativa do modelo

Esse modelo apresentou o menor erro médio e maior estabilidade entre os folds de validação temporal, mostrando-se mais eficaz que o Random Forest no ajuste à variabilidade interanual da produtividade.

---

## 6. Resultados e Métricas

- **Teste (2021–2025)**:
  - RMSE: 0.78 t/ha
  - R²: 12.50 
  - MAE: 0.75 t/ha


---

## 7. Vídeo de Demonstração

- Link (não listado): [**https://youtu.be/VideoDemonstracao**]([[https://youtu.be/fazerainda](https://youtu.be/PvSzy-tWDuI)](https://youtu.be/PvSzy-tWDuI))

---

### Autores

- Grupo Sprint 2 – Ingredion Challenge

- Matheus Augusto Rodrigues Maia_RM560683
- Alex da Silva Lima_RM559784
- Johnatan Sousa Macedo Loriano _RM559546
- Bruno Henrique Nielsen Conter_RM560518 
- Fabio Santos Cardoso_RM560479

---
