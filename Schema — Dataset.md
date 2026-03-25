# 📊 Schema — Dataset Folha-Dourada (V1.0)

**Unidade de análise:** 1 linha = 1 grid (90m x 90m)
**Objetivo:** alimentar modelo de triagem aurífera multi-sinal
**Proporção alvo:** 1 positivo : 3 negativos difíceis

---

## 🧭 1. Identificação do ponto

| Coluna      | Tipo   | Fonte          | Descrição                   |
| ----------- | ------ | -------------- | --------------------------- |
| grid_id     | string | gerado         | identificador único do grid |
| latitude    | float  | centro do grid | coordenada do centro        |
| longitude   | float  | centro do grid | coordenada do centro        |
| data_imagem | date   | Landsat SR     | data da imagem utilizada    |
| month       | int    | derivado       | mês da imagem (1–12)        |

---

## 🌿 2. Vegetação — Landsat Surface Reflectance

| Coluna    | Tipo  | Fonte      | Descrição                             |
| --------- | ----- | ---------- | ------------------------------------- |
| ndvi_mean | float | Landsat SR | média do NDVI no grid                 |
| ndvi_std  | float | Landsat SR | desvio padrão (textura)               |
| evi_mean  | float | Landsat SR | índice EVI (melhor em floresta densa) |
| ndwi_mean | float | Landsat SR | índice de umidade foliar              |
| savi_mean | float | Landsat SR | vegetação ajustada ao solo            |

---

## 🏔️ 3. Topografia — SRTM

| Coluna        | Tipo  | Fonte | Descrição                  |
| ------------- | ----- | ----- | -------------------------- |
| altitude_mean | float | SRTM  | altitude média do grid (m) |
| slope_mean    | float | SRTM  | declividade média (graus)  |
| slope_std     | float | SRTM  | variação local de relevo   |

---

## 🌊 4. Hidrografia

| Coluna       | Tipo  | Fonte            | Descrição                         |
| ------------ | ----- | ---------------- | --------------------------------- |
| dist_rio_m   | float | Hidrografia IBGE | distância ao rio mais próximo (m) |
| log_dist_rio | float | derivado         | log(dist_rio_m + 1)               |

---

## 🪨 5. Geologia — CPRM

| Coluna     | Tipo        | Fonte | Descrição                             |
| ---------- | ----------- | ----- | ------------------------------------- |
| geo_classe | int (0/1/2) | CPRM  | 0=desfavorável, 1=neutra, 2=favorável |

---

## 📐 6. Normalização espacial (aplicada após coleta)

| Coluna   | Tipo  | Fonte     | Descrição                                     |
| -------- | ----- | --------- | --------------------------------------------- |
| *_zscore | float | calculado | z-score por região para cada feature numérica |

**Regra:**
Normalizar **por região geográfica consistente** (ex: blocos de 50km ou bacias hidrográficas)

---

## 🎯 7. Target — variável resposta

| Coluna | Tipo      | Fonte           | Descrição                                    |
| ------ | --------- | --------------- | -------------------------------------------- |
| ouro   | int (0/1) | SIGMINE + campo | 1 = garimpo confirmado, 0 = negativo difícil |

---

## ⚖️ 8. Regras críticas do dataset

### 🔴 Negativos

* Devem ser:

  * próximos de rios
  * mesma geologia
  * mesma região
* Nunca usar negativos aleatórios

---

### 🔴 Imagens

* Landsat **Surface Reflectance (SR)**
* Nuvem < 10%
* Mesma época do ano
* Aplicar máscara de:

  * nuvem
  * sombra (QA band)

---

### 🔴 Unidade espacial

* Grid fixo: **90m x 90m (3x3 pixels Landsat)**

---

### 🔴 Balanceamento

* Proporção alvo:

  * 1 positivo : 3 negativos

---

### 🔴 Normalização

* Sempre aplicar **z-score por região**
* Nunca usar valores absolutos entre regiões

---

## 🧠 Observações estratégicas

* Dataset define o sucesso do projeto
* Modelo é secundário
* Features espaciais (std, slope_std) são diferenciais
* O sistema aprende **padrão combinado**, não variável isolada

---

## 📦 Versão

**Versão:** V1.0
**Status:** pronto para implementação
**Próxima revisão:** após primeiro teste crítico (Etapa 5)

---
