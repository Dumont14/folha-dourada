# Projeto Folha-Dourada — Roteiro Definitivo V4

## Objetivo

Construir um sistema de triagem que identifica áreas com alta similaridade geoespacial e espectral a regiões onde ouro já foi encontrado, reduzindo drasticamente o custo de prospecção de campo.

---

## Hipótese Operacional

Ouro não é detectado diretamente.

O sistema funciona assim:

> múltiplos sinais fracos + coerência geológica + contexto espacial → padrão detectável por ML

---

## Etapa 1 — Ambiente e Contas
**Tempo estimado:** 1 dia  
**Custo:** zero

### Instalar
- Python >= 3.10
- QGIS

### Criar contas gratuitas
- Google Colab
- USGS EarthExplorer
- SIGMINE / ANM

---

## Etapa 2 — Coleta de Dados
**Tempo estimado:** 1 a 2 semanas  
**Custo:** zero

### Imagens de satélite
- **Fonte:** Landsat Surface Reflectance (SR)
- **Regras obrigatórias:**
  - Usar Surface Reflectance (não Top of Atmosphere)
  - Cobertura de nuvem < 10%
  - Mesma época do ano para todos os pontos (consistência temporal)
  - Aplicar máscara de nuvem e sombra de nuvem via QA band

### Dados auxiliares (todos gratuitos)
- **SRTM:** altitude e slope (declividade)
- **Hidrografia:** shapefiles de rios (IBGE)
- **Geologia:** mapa geológico vetorial (CPRM)

---

## Etapa 3 — Construção do Dataset
**Tempo estimado:** 2 a 4 semanas  
**Esta é a etapa mais importante do projeto.**

### Unidade de análise
- Grid de **3x3 pixels = 90m x 90m**
- Para cada grid extrair: média e desvio padrão de cada feature

### Definição dos rótulos

**Positivos (ouro = 1)**
- Garimpos confirmados no SIGMINE
- Filtrados e validados com conhecimento local

**Negativos difíceis (ouro = 0)**
- DEVEM ser: próximos de rios, mesma geologia, mesma região
- NUNCA usar negativos aleatórios
- Proporção alvo: 1 positivo para 3 negativos difíceis

### Features finais

| Feature | Índice | Descrição |
|---|---|---|
| ndvi_mean | NDVI | média no grid |
| ndvi_std | NDVI | desvio padrão (textura) |
| evi_mean | EVI | não satura em floresta densa |
| ndwi_mean | NDWI | umidade foliar |
| savi_mean | SAVI | vegetação corrigida para solo |
| altitude_mean | SRTM | altitude média |
| slope_mean | SRTM | declividade média |
| slope_std | SRTM | variação local de relevo |
| dist_rio_m | Hidrografia | distância ao rio mais próximo (metros) |
| geo_classe | CPRM | 0=desfavorável / 1=neutra / 2=favorável |

---

## Etapa 3.5 — Normalização Espacial
**Obrigatório — sem isso o modelo falha em regiões novas.**

- Aplicar z-score por região em todas as features numéricas
- Trabalhar com valores relativos, não absolutos
- Gerar colunas `*_zscore` para cada feature normalizada

---

## Etapa 4 — Modelagem
**Tempo estimado:** 1 semana  
**Seguir esta ordem obrigatoriamente. Cada modelo deve superar o anterior.**

### 1. Baseline heurístico (referência mínima)
- Regra simples: score = proximidade ao rio + geologia favorável
- Se o ML não superar isso, o ML não presta

### 2. Random Forest
- Validar se o modelo aprende algo real além do heurístico

### 3. XGBoost (modelo principal do MVP)
- Melhor performance com dados tabulares
- Usar `class_weight` para lidar com desbalanceamento

### Proibido nesta fase
- Deep learning
- CNN
- Qualquer modelo complexo

---

## Etapa 5 — Teste Crítico
**Este é o divisor de águas do projeto.**

### Regra de ouro
- Treinar na **região A**
- Testar na **região B** (separação geográfica real)
- Nunca validação aleatória

### Métricas corretas
- `precision@top10%` — dos 10% melhor ranqueados, quantos têm ouro?
- `recall aurífero` — quantas áreas com ouro foram encontradas?
- Curva de ganho acumulado

### Não usar
- Accuracy — o problema é de **ranking**, não de classificação

### Interpretação
- **Funciona:** modelo generaliza → projeto válido → seguir para etapa 6
- **Falha:** voltar para **etapa 3** (nunca para etapa 4 — o problema é sempre o dataset)

---

## Etapa 6 — Mapa de Calor Aurífero
**Se a etapa 5 funcionar, este é o produto.**

- Score de 0 a 1 por grid
- Visualização como heatmap no QGIS
- Entrega: ranking de coordenadas prioritárias ordenadas por score

> **Atenção:** o score representa **similaridade com padrões históricos**, não probabilidade real de ouro.

---

## Etapa 7 — Validação em Campo
**Sem esta etapa, o projeto não existe.**

- Visitar os pontos de maior score
- Confirmar ou refutar com prospecção real
- Adicionar resultados ao dataset
- Re-treinar o modelo
- Repetir — a precisão aumenta a cada ciclo

---

## Schema do Dataset

**1 linha = 1 grid de 90x90m**

| Coluna | Tipo | Fonte | Descrição |
|---|---|---|---|
| grid_id | string | gerado | identificador único |
| latitude | float | centroide | coordenada centro |
| longitude | float | centroide | coordenada centro |
| data_imagem | date | Landsat | data da imagem usada |
| ndvi_mean | float | Landsat SR | média NDVI no grid |
| ndvi_std | float | Landsat SR | desvio padrão NDVI |
| evi_mean | float | Landsat SR | EVI (não satura em floresta densa) |
| ndwi_mean | float | Landsat SR | umidade foliar |
| savi_mean | float | Landsat SR | vegetação corrigida para solo |
| altitude_mean | float (m) | SRTM | altitude média |
| slope_mean | float (graus) | SRTM | declividade média |
| slope_std | float | SRTM | variação local de relevo |
| dist_rio_m | float (m) | Hidrografia IBGE | distância ao rio mais próximo |
| geo_classe | int (0/1/2) | CPRM | 0=desfav / 1=neutra / 2=favorável |
| ouro | int (0 ou 1) | SIGMINE + campo | target — 1=confirmado / 0=negativo difícil |
| *_zscore | float | calculado | z-score por região de cada feature |

**Total: 16 colunas base + colunas zscore**  
**Proporção alvo: 1 positivo para 3 negativos difíceis**

---

## Stack Completa

| Categoria | Ferramenta |
|---|---|
| Linguagem | Python >= 3.10 |
| Visualização geoespacial | QGIS |
| Processamento em nuvem | Google Colab |
| Imagens de satélite | Landsat SR — USGS EarthExplorer |
| Relevo | SRTM |
| Geologia | CPRM |
| Garimpos | SIGMINE / ANM |
| Rasters | Rasterio |
| Vetores e geo | GeoPandas |
| ML | Scikit-learn, XGBoost |
| Visualização | Matplotlib |

---

## Custo

| Fase | Custo |
|---|---|
| Etapas 1 a 6 | Zero |
| Etapa 7 | Deslocamento + operação de campo |

---

## Regras de Ouro do Projeto

1. Se falhar, o problema é o **dataset** — nunca o modelo
2. Os **negativos difíceis** definem a qualidade do sistema
3. **Generalização geográfica** é o único teste que importa
4. O objetivo é **priorização**, não previsão perfeita
5. Sem **validação de campo**, o projeto não existe

---

## Resumo em Uma Linha

> Você não está tentando encontrar ouro.  
> Você está tentando **reduzir a incerteza de onde procurar**.

---

*Projeto desenvolvido com apoio de Claude (Anthropic) e outra IA — briga incluída.*
