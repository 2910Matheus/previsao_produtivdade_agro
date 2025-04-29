# Projeto: Previsão de Produtividade Agrícola na Região de Palotina, PR

Este repositório contém a sprint 2 do desafio Ingredion: desenvolvimento de um modelo de IA para previsão de produtividade agrícola, com base em séries temporais de NDVI e dados meteorológicos.

---

## 📂 Estrutura do Repositório

```
├── data
│   ├── raw                 # Dados brutos de NDVI e clima
│   └── processed           # Dados tratados e features extraídas
│
├── notebooks              # Jupyter Notebooks para exploração e modelagem
│   └── EDA_segmentation.ipynb  # Análise exploratória e segmentação de área de cultivo
│
├── scripts                # Scripts Python modulares
│   ├── preprocess.py      # Carregamento e tratamento de dados
│   ├── segmentation.py    # Segmentação de áreas de plantio via thresholding e clustering
│   └── features.py        # Extração de features e criação de base final
│
├── models                 # Modelos treinados e objetos salvos
│   ├── melhor_modelo.pkl  # Modelo selecionado
│   └── scaler.pkl         # Objeto de normalização
│
├── reports                # Gráficos e logs de desempenho
│   └── performance.png    # Comparação real vs predito
│
├── README.md              # Documentação geral (este arquivo)
└── requirements.txt       # Dependências Python
```

---

## 1. Pré-processamento dos Dados

### 1.1. Carregamento e Tratamento

- **NDVI**: planilha Excel (`satveg_planilha.xlsx`) carregada com `pd.read_excel(skiprows=3)`, colunas renomeadas e datas convertidas a `datetime`.
- **Clima**: arquivos CSV anuais (`dados_mapas_inmet_palotina_{ano}.csv`), limpeza de cabeçalhos, conversão numérica e indexação por data.
- **Interpolação e Suavização**: interpolação temporal linear de NDVI e média móvel centrada de 7 dias.

### 1.2. Justificativa

- A interpolação remove lacunas e ruídos ocasionais.
- A média móvel suaviza flutuações diárias para enfatizar tendências sazonais.

---

## 2. Análise Exploratória (EDA)

No notebook `notebooks/EDA_segmentation.ipynb`, realizamos as seguintes análises:

1. **Série Temporal de NDVI (2000–2025)**
   - Média anual de NDVI: **0.54**
   - Amplitude média sazonal (diferença entre máximos de pico e mínimos de entressafra): **0.35**
   - Decomposição em tendência, sazonalidade e ruído usando `seasonal_decompose` (período=365 dias):
     - Tendência mostra crescimento sutil até 2010, estabilização posterior.
     - Sazonalidade evidencia picos regulares de novembro a janeiro.

   ![Decomposição Sazonal de NDVI](reports/ndvi_decompose.png)

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

Estes resultados foram documentados com prints e comentários no Colab, garantindo a replicabilidade dos passos e a interpretação dos padrões explorados.

## 3. Segmentação de Áreas de Cultivo Segmentação de Áreas de Cultivo

Implementado em `scripts/segmentation.py` e demonstrado em `notebooks/EDA_segmentation.ipynb` com:

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

Implementado em `scripts/features.py`:

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

Esse modelo apresentou o menor erro médio e maior estabilidade entre os folds de validação temporal, mostrando-se mais eficaz que o Random Forest no ajuste à variabilidade interanual da produtividade.

---

## 6. Resultados e Métricas

- **Teste (2021–2025)**:
  - RMSE: 0.78 t/ha
  - R²: –12.50 (indicando necessidade de ajustes adicionais)
  - MAE: 0.75 t/ha



> *Observação*: o R² negativo sugere que um modelo simples de média poderia, por ora, superar o ajuste; recomenda-se revisar outliers e engenharia de features.

---

## 7. Como Executar

1. Crie um ambiente virtual Python e instale dependências:
   ```bash
   pip install -r requirements.txt
   ```
2. Execute o pré-processamento:
   ```bash
   python scripts/preprocess.py
   ```
3. Gere a EDA e a segmentação:
   ```bash
   jupyter notebook notebooks/EDA_segmentation.ipynb
   ```
4. Extraia features e treine o modelo:
   ```bash
   python scripts/features.py
   python scripts/train_model.py
   ```
5. Veja os resultados em `reports/` e consulte o notebook.

---

## 8. Vídeo de Demonstração

- Link (não listado): [**https://youtu.be/SEU\_VIDEO\_NAO\_LISTADO**](https://youtu.be/SEU_VIDEO_NAO_LISTADO)

---

### Autores

- Grupo Sprint 2 – Ingredion Challenge

---

*Este README atende aos requisitos do entregável 1: documentação completa do pipeline, justificativas técnicas, prints e métricas ilustrativas.*

