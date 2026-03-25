# ⚙️ Pipeline de Geração do Dataset — Folha-Dourada (V1.0)

**Objetivo:** gerar dataset estruturado pronto para ML a partir de dados públicos
**Entrada:** coordenadas SIGMINE + imagens Landsat + dados auxiliares
**Saída:** tabela final (CSV/Parquet) com features e target

---

# 🧭 VISÃO GERAL DO PIPELINE

1. Definir área de estudo
2. Criar grid espacial
3. Coletar imagens Landsat (SR + QA)
4. Calcular índices espectrais
5. Extrair features por grid
6. Integrar dados auxiliares (SRTM, rios, geologia)
7. Gerar rótulos (positivo / negativo difícil)
8. Normalizar dados
9. Exportar dataset final

---

# 🧱 ETAPA 1 — Definir área de estudo

### Input:

* shapefile da região (ex: Tapajós)

### Output:

* bounding box / polígono da área

```python
import geopandas as gpd

area = gpd.read_file("area_estudo.shp")
```

---

# 🔲 ETAPA 2 — Criar grid (90m x 90m)

### Estratégia:

* gerar grid regular sobre a área

```python
import numpy as np
from shapely.geometry import box

def create_grid(area, cell_size=90):
    minx, miny, maxx, maxy = area.total_bounds
    grid_cells = []

    for x in np.arange(minx, maxx, cell_size):
        for y in np.arange(miny, maxy, cell_size):
            grid_cells.append(box(x, y, x+cell_size, y+cell_size))

    return gpd.GeoDataFrame(geometry=grid_cells, crs=area.crs)
```

---

# 🛰️ ETAPA 3 — Coletar Landsat (Surface Reflectance)

### Regras:

* coleção SR (Surface Reflectance)
* nuvem < 10%
* mesma época do ano

### Opção recomendada:

👉 usar Google Earth Engine (mais eficiente)

---

# 🌿 ETAPA 4 — Calcular índices espectrais

### Índices:

```python
NDVI = (NIR - RED) / (NIR + RED)
NDWI = (GREEN - NIR) / (GREEN + NIR)
SAVI = ((NIR - RED) / (NIR + RED + 0.5)) * 1.5
EVI  = 2.5 * (NIR - RED) / (NIR + 6*RED - 7.5*BLUE + 1)
```

---

# 📦 ETAPA 5 — Extrair features por grid

### Para cada grid:

* média dos índices
* desvio padrão (textura)

```python
def extract_stats(raster, geometry):
    values = raster.sample(geometry)
    return {
        "mean": np.mean(values),
        "std": np.std(values)
    }
```

---

# 🏔️ ETAPA 6 — Topografia (SRTM)

* extrair:

  * altitude_mean
  * slope_mean
  * slope_std

```python
import richdem as rd

dem = rd.LoadGDAL("srtm.tif")
slope = rd.TerrainAttribute(dem, attrib='slope_degrees')
```

---

# 🌊 ETAPA 7 — Hidrografia

### Calcular distância ao rio:

```python
from shapely.ops import nearest_points

def dist_to_river(point, rivers):
    nearest = rivers.distance(point).min()
    return nearest
```

### Feature derivada:

```python
log_dist = np.log(dist_rio_m + 1)
```

---

# 🪨 ETAPA 8 — Geologia

### Interseção espacial:

```python
grid["geo_classe"] = gpd.sjoin(grid, geologia)["classe"]
```

### Mapear:

* 0 = desfavorável
* 1 = neutra
* 2 = favorável

---

# 🎯 ETAPA 9 — Gerar rótulos

## Positivos:

* pontos SIGMINE

```python
grid["ouro"] = 0
grid.loc[grid.intersects(garimpos), "ouro"] = 1
```

---

## Negativos difíceis:

Filtrar grids que:

* estão próximos de rios
* mesma geologia
* não possuem garimpo

---

# ⚖️ ETAPA 10 — Balanceamento

```python
positivos = df[df.ouro == 1]
negativos = df[df.ouro == 0].sample(len(positivos)*3)

df_final = pd.concat([positivos, negativos])
```

---

# 📐 ETAPA 11 — Normalização espacial

### Definir região (ex: blocos ou clusters)

```python
df["ndvi_z"] = (df["ndvi_mean"] - df.groupby("regiao")["ndvi_mean"].transform("mean")) / df.groupby("regiao")["ndvi_mean"].transform("std")
```

---

# 💾 ETAPA 12 — Exportar dataset

```python
df_final.to_parquet("dataset.parquet")
```

---

# 🧪 CHECKLIST DE QUALIDADE

Antes de usar o dataset:

* [ ] Sem valores nulos
* [ ] Distribuição balanceada
* [ ] Features normalizadas
* [ ] Sem vazamento de informação
* [ ] Coordenadas corretas

---

# ⚠️ ERROS PROIBIDOS

* usar imagens sem correção atmosférica
* misturar estações do ano
* usar negativos aleatórios
* validar com split aleatório
* pular normalização espacial

---

# 🎯 RESULTADO FINAL

Dataset estruturado com:

* features geoespaciais
* target confiável
* pronto para Random Forest / XGBoost

---

# 🔁 CICLO

1. gerar dataset
2. treinar modelo
3. testar em nova região
4. ajustar dataset

---

## 📌 REGRA FINAL

> Se o modelo falhar, o erro está no dataset — não no algoritmo.

---
