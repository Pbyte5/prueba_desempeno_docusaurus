---
title: Recomendación para el equipo de producto
sidebar_position: 4
---

# Recomendación para el equipo de producto

**Hallazgo.** Los productos **`23843` (PAPER CRAFT , LITTLE BIRDIE)** y **`23166` (MEDIUM CERAMIC TOP STORAGE JAR)** (consulta P7) generaron en conjunto **$250,170.52** de revenue bruto pero perdieron **$245,949.24** (**98.31%**) por cancelaciones y devoluciones, frente al **17.73%** y **3.25%** en promedio para sus categorías **Papelería** y **Hogar** (consulta P2b). Asimismo, en artículos de venta minorista recurrente destacan **`23113` (PANTRY CHOPPING BOARD)** con **$4,803.06** perdidos (**81.84%** de su revenue) y **`21108` (FAIRY CAKE FLANNEL ASSORTED COLOUR)** con **$6,591.42** perdidos (**36.50%**).

**Recomendación concreta.** Revisar calidad, ficha técnica y proveedor de los **9** productos del Top 20 de la lista P7 con tasa de devolución superior al **20%** (que concentran **$270,792.59** en devoluciones con una tasa ponderada del **84.11%**), empezando por **`23843`** y **`23166`** (para auditar transacciones mayoristas o cancelaciones operativas anómalas) y continuando con **`23113`**, **`23056`** y **`21108`**. Además, unificar la descripción de los **207** códigos con escrituras inconsistentes (consulta P6a/P6b) que concentran **$1,111,222.04** de revenue bruto en el catálogo comercial. Meta propuesta: bajar la tasa de devolución de estos 9 productos críticos del **84.11%** al **20.00%**, lo que recuperaría cerca de **$206,400.00** de revenue neto en el periodo analizado (o reducir la tasa de devolución global de la compañía del **4.64%** al **3.00%**, recuperando cerca de **$168,000.00** de revenue neto).

**Respaldo.** Consultas P2b, P3b, P6a y P7 · tabla `fact_daily_product` · filas rechazadas relevantes: **5,990 filas** por código no perteneciente al catálogo (`NON_CATALOG_CODE`, correspondientes a 2,995 códigos únicos) y **4,970 filas** con precio unitario no positivo (`NON_POSITIVE_PRICE`), sobre un total auditado de **552,648 registros rechazados** (`rejected_records`).

---

## Métricas clave del análisis

### 1. Top productos con mayor impacto por devoluciones (Consulta P7)

| Código | Descripción | Categoría | Revenue Bruto | Monto Devolución | % Devolución Revenue | Variantes Descripción |
| :--- | :--- | :--- | :---: | :---: | :---: | :---: |
| **`23843`** | PAPER CRAFT , LITTLE BIRDIE | Papelería | $168,469.60 | $168,469.60 | **100.00%** | 1 |
| **`23166`** | MEDIUM CERAMIC TOP STORAGE JAR | Hogar | $81,700.92 | $77,479.64 | **94.83%** | 1 |
| **`22423`** | REGENCY CAKESTAND 3 TIER | Hogar | $174,156.54 | $9,697.05 | **5.57%** | 1 |
| **`85123A`** | WHITE HANGING HEART T-LIGHT HOLDER | Hogar | $106,415.23 | $6,624.30 | **6.22%** | 2 |
| **`21108`** | FAIRY CAKE FLANNEL ASSORTED COLOUR | Hogar | $18,056.28 | $6,591.42 | **36.50%** | 1 |
| **`23113`** | PANTRY CHOPPING BOARD | Hogar | $5,868.50 | $4,803.06 | **81.84%** | 1 |
| **`48185`** | DOORMAT FAIRY CAKE | Hogar | $23,872.91 | $4,554.90 | **19.08%** | 1 |
| **`21175`** | GIN + TONIC DIET METAL SIGN | Hogar | $27,029.97 | $3,775.33 | **13.97%** | 2 |
| **`47566B`** | TEA TIME PARTY BUNTING | Hogar | $12,926.42 | $3,692.95 | **28.57%** | 1 |
| **`22273`** | FELTCRAFT DOLL MOLLY | Hogar | $12,001.59 | $3,512.65 | **29.27%** | 1 |

### 2. Proporción de devoluciones por categoría (Consulta P2b)

| Categoría | Revenue Bruto | Monto Devolución | % Devolución Revenue | % Devolución Unidades |
| :--- | :---: | :---: | :---: | :---: |
| **Papelería** | $1,032,160.52 | $182,976.37 | **17.73%** | 9.99% |
| **Hogar** | $8,830,828.08 | $287,314.05 | **3.25%** | 3.89% |
| **Electrónica** | $275,043.13 | $4,372.06 | **1.59%** | 1.48% |
| **Ropa** | $76,592.89 | $1,139.61 | **1.49%** | 0.91% |
| **Deportes** | $31,644.40 | $99.07 | **0.31%** | 0.37% |

### 3. Inconsistencias de catálogo y calidad de datos (Consultas P6a, P6b y Auditoría)

* **207 códigos** poseen variantes en la descripción tal como fueron recibidas (`n_raw_description_variants > 1`) y persisten tras convertir a mayúsculas.
* Estos 207 códigos concentran **$1,111,222.04** de revenue bruto (**$1,079,451.64** netos).
* Se rechazaron **5,990 registros** por no pertenecer al catálogo comercial activo (`NON_CATALOG_CODE`), evitando contaminar la dimensión de producto.
* Se rechazaron **4,970 registros** con precio unitario menor o igual a cero (`NON_POSITIVE_PRICE`).
