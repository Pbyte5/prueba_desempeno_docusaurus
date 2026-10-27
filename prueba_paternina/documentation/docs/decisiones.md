---
title: Decisiones técnicas
sidebar_position: 2
---

# Documento de decisiones técnicas

## 1. Arquitectura

```
data/raw/*.csv ─> extract_history_sales / extract_daily_sales ─> staging (parquet)
                    ─> validate_and_transform ─> load_to_db ─> run_quality_check
```

- **Airflow 3.3.2** con `LocalExecutor`, todo en Docker (`docker compose up`). Servicios: `airflow-apiserver`
  (UI y API en `localhost:8080`; en Airflow 3 reemplaza al "webserver"), `airflow-scheduler`, `airflow-dag-processor`
  (en Airflow 3 es un proceso aparte), `postgres-airflow` (metadatos) y `postgres-dw` (repositorio analítico, separado).
- **TaskFlow API** (`@dag`, `@task`, imports de `airflow.sdk`). El DAG solo orquesta; la lógica vive en
  `src/prueba_analitica/pipeline/scripts/` (`extract`, `transform`, `load`, `data_quality`), una etapa por archivo.
- **Staging en parquet** entre tareas: los DataFrames no viajan por XCom (es para datos pequeños), solo rutas y conteos.
- **Connection y Variables por variables de entorno** (`AIRFLOW_CONN_DATAMART`, `AIRFLOW_VAR_DATAMART_*`): quedan
  listas al levantar el compose, sin entrar a la UI. Ningún usuario, contraseña ni cadena de conexión está en el
  código: todo sale de `.env` (excluido por `.gitignore`; la plantilla es `.env.example`).
  - Connection: `datamart`.
  - Variables: `datamart_history_files`, `datamart_daily_files`, `datamart_source_priority`,
    `datamart_source_tz` y `datamart_products_api_url`.
- **API de productos:** implementada como servicio FastAPI en Docker Compose. Airflow consulta el catálogo antes de
  transformar los datos; las categorías persistidas tienen prioridad y las reglas CSV clasifican los códigos nuevos.

## 2. Exploración de las fuentes

Cada tarea `extract_*` escribe en su log la exploración de su fuente: filas, nulos por columna, cantidades y precios
≤ 0, filas totalmente duplicadas, y códigos y descripciones distintos. El rango de fechas, los motivos de rechazo y
los conteos finales salen en el resultado (XCom) de `validate_and_transform`. Todo se lee en la UI de Airflow
(clic en la tarea → *Logs* / *XCom*). Los rechazos también quedan consultables en `rejected_records` (consulta
*Auditoría* de [Consultas SQL](./consultas)).

Problemas de calidad (los cuatro primeros los anticipa el enunciado) y qué decidí hacer con cada uno:

| Problema | Decisión |
|---|---|
| Cantidades ≤ 0 | Se separan de las ventas: van a `fact_returns`. |
| Códigos que no siguen el catálogo (POST, M, D, DOT, TEST001…) | Se rechazan (`NON_CATALOG_CODE`). |
| Descripciones con mayúsculas/minúsculas mezcladas | Se normalizan y se elige un nombre canónico (sección 4). |
| Sin customer ID | Se incluyen, con `customer_id` nulo y `is_guest = TRUE` (sección 4). |
| Precio ≤ 0 | Se rechaza (`NON_POSITIVE_PRICE`). |
| Fechas en formatos distintos entre fuentes | Se parsean con formatos conocidos y se pasan a UTC; si no encajan, se rechazan (`INVALID_INVOICE_DATE`). |
| Nombres de columna distintos entre fuentes | Se unifican al leer (`Invoice`/`InvoiceNo`, `Price`/`UnitPrice`, `Customer ID`/`CustomerID`). |
| Filas duplicadas (dentro de una fuente y entre fuentes) | Se rechazan con su motivo (sección 4). |

> **COMPLETAR antes de entregar** con las cifras reales de tu ejecución (log de las tareas `extract_*`, XCom de
> `transform_and_validate` y la consulta *Auditoría*): filas por fuente `___`/`___`, rango de fechas `___`,
> nulos de customer ID `___` %, cantidades ≤ 0 `___`, precios ≤ 0 `___`, códigos distintos `___`, solape entre
> fuentes `___` filas. El código y las consultas ya producen estos números; falta pegarlos aquí.

## 3. Modelo del repositorio analítico

Una dimensión, tres tablas de hechos y un log de rechazos. Está pensado para las preguntas de negocio y para
la separación entre ventas y devoluciones. El diagrama se muestra a continuación.

![Diagrama del modelo de datos analítico](./img/modelo-datos.png)

| Tabla | Para qué |
|---|---|
| `fact_sales` | Una fila por línea de venta válida (`quantity > 0` y `unit_price > 0`). Guarda `gross_revenue`. |
| `fact_returns` | Devoluciones y ajustes (`quantity <= 0`). `movement_type` = `RETURN` si la factura empieza con `C` (nota crédito) o `ADJUSTMENT` si no. |
| `fact_daily_product` | Revenue bruto, devoluciones y **revenue neto por producto y por día**, calculado y almacenado como pide la regla de negocio. |
| `dim_product` | Una fila por código: nombre canónico, categoría, y cuántas variantes de descripción tuvo. |
| `rejected_records` | Cada fila rechazada, con fuente, posición en su archivo, motivo y valores originales. |

Por qué así:
- **Ventas y devoluciones en tablas separadas:** el neto se calcula sin mezclar signos y la tasa de devolución sale directa.
- **`fact_daily_product` es una tabla y no una vista:** el enunciado pide "calcular y almacenar" el revenue neto. Se
  construye con SQL dentro de la misma transacción de carga y `run_quality_checks` verifica que cuadre con los hechos.
- **Sin dimensión de clientes, países ni fechas:** ninguna pregunta necesita atributos extra. `customer_id` + `is_guest`
  y `country` (pocos valores) viven en el hecho, y `date_trunc` sobre `invoice_date` resuelve los periodos. Menos tablas, menos joins.
- **Llave natural `stock_code`** en `dim_product`: ya viene normalizada, así que no hace falta una llave sustituta.

## 4. Casos ambiguos (sección 5 del enunciado)

| Caso | Decisión | Justificación e impacto |
|---|---|---|
| **Transacciones sin customer ID** | Se **incluyen** con `customer_id = NULL` e `is_guest = TRUE`. | Excluirlas borra ventas reales y distorsiona los totales de revenue. Incluirlas permite además responder la pregunta 5 (comparar identificados vs. no identificados). Impacto: consulta **P5b** (qué % de líneas y de revenue representan) y **P5a** (cómo se comportan). |
| **Nombre canónico de un producto** | Se normaliza (espacios colapsados + MAYÚSCULAS) y el nombre canónico es la **variante normalizada más frecuente** del código; en empate, la primera en orden alfabético. | Criterio objetivo y reproducible (mismos datos → mismo nombre). `dim_product` conserva `n_raw_description_variants` (escrituras tal como llegaron: `CANDLE`, `Candle`…) y `n_description_variants` (tras pasar a mayúsculas; si es > 1, la diferencia no es solo de mayúsculas). |
| **Solape entre las dos fuentes** | Se usa una **clave compuesta**: factura + producto + cantidad + precio + fecha UTC + cliente. Si una fila ya apareció, se descarta la repetida y se registra como `DUPLICATE_CROSS_SOURCE`. Gana la fuente con prioridad, que se define en la Variable `datamart_source_priority` (por defecto `history,daily`). | No se descarta una fuente entera: solo lo que realmente coincide. `history` gana por ser el archivo histórico consolidado; cambiarlo es cambiar una Variable. |
| **Duplicados dentro de una misma fuente** | Misma clave; se conserva la primera y las demás se registran como `DUPLICATE_WITHIN_SOURCE`. | Filas idénticas en factura, producto, cantidad, precio, minuto y cliente se tratan como doble registro. Si el negocio las considerara líneas legítimas, bastaría agregar un número de línea a la clave. |
| **Códigos que no son de catálogo** | Se **rechazan**. Es código válido el que cumple `^\d{5}[A-Z]{0,2}$` (85123A). | POST, M, D, DOT, BANK CHARGES, TEST001… son envíos, descuentos, comisiones o pruebas, no productos. Dejarlos contaminaría los rankings. Costo: se pierde algún producto real con código atípico; los códigos rechazados se pueden auditar (consulta P6c y *Auditoría*). |
| **Cantidad ≤ 0** | Va a `fact_returns` (sección 3). | Regla del negocio. |
| **Precio ≤ 0** | Se rechaza (`NON_POSITIVE_PRICE`) en ventas y en devoluciones. | Regla del negocio. Los ajustes de inventario con precio 0 no tienen valor monetario: quedan en el log, no en `fact_returns`. |
| **Fechas** | Se interpretan en la zona horaria `Europe/London` (Variable `datamart_source_tz`) y se pasan a **UTC**. Nunca se asume día/mes (es ambiguo): `data.csv` se lee como mes/día/año y el otro archivo como año-mes-día; lo que no encaje se rechaza. | Las fuentes son de una tienda del Reino Unido. Es un supuesto que habría que confirmar con el negocio. |
| **País** | `EIRE → Ireland`, `RSA → South Africa`, `USA → United States`; vacío → `Unspecified`. | Evita que un mismo país aparezca con dos nombres en la pregunta 4. |
| **Moneda** | No se convierte: los montos quedan en la moneda de origen (libras). | Los datasets no traen la moneda por fila ni tasas de cambio; convertir exigiría inventarlas. |
| **Categoría** | Palabras clave por prioridad (Electrónica, Deportes, Papelería, Ropa); si ninguna coincide, `Hogar`. `category_source` indica si fue por regla o por defecto. | Es una heurística explícita y editable en `seeds/category.csv`. El catálogo real es en su mayoría decoración y artículos del hogar, por eso es la categoría por defecto. |

Toda fila de entrada termina en exactamente un lugar: venta, devolución o rechazo con motivo. `run_quality_checks`
lo verifica (`filas de entrada = ventas + devoluciones + rechazos`).

## 5. Reglas del negocio y dónde están en el código

| Regla (sección 5) | Dónde |
|---|---|
| Cantidad ≤ 0 → devolución o ajuste, separada de las ventas | `src/prueba_analitica/pipeline/scripts/transform.py` |
| Precio unitario > 0 en una venta válida; si no, al log de rechazos con motivo | `transform.py`, validación de filas |
| Fechas en UTC | `transform.py`, conversión a UTC |
| Códigos en mayúsculas y sin espacios | `transform.py`, estandarización |
| Revenue bruto y revenue neto por producto y día, almacenados | Transformación y carga al almacén |

## 6. Idempotencia

`load` ejecuta **en una sola transacción**: `TRUNCATE … RESTART IDENTITY` de todas las tablas → carga con
`COPY` de la dimensión, los hechos y los rechazos → construcción de `fact_daily_product` → `COMMIT`. Resultado:

- Correr el DAG dos veces con los mismos datos deja **exactamente el mismo estado** (lo comprobé comparando un checksum
  de `fact_sales` y los conteos de todas las tablas entre dos ejecuciones).
- Si algo falla a mitad de carga hay `ROLLBACK`: nunca quedan datos parciales, así que los reintentos (`retries=2`) son seguros.
- Los archivos de staging se sobrescriben en cada ejecución, y la transformación es determinista (mismo orden, mismos desempates).

Compromiso: es un *full refresh*. Es correcto aquí porque las dos fuentes son volcados completos. Con cargas incrementales
reales habría que particionar por fecha (`DELETE WHERE invoice_date = …` + `INSERT`) o usar `ON CONFLICT DO NOTHING`.

## 7. Otras decisiones

- `schedule="@daily"`, `catchup=False`, `max_active_runs=1`, `retries=2` con espera de 1 minuto.
- El DAG arranca **activo** (`DAGS_ARE_PAUSED_AT_CREATION=false`) para que corra sin intervención al levantar el entorno.
- **Datos de entrada:** los CSV de entrada van en `data/raw/`; sus rutas relativas se configuran con
  `DATAMART_HISTORY_FILE` y `DATAMART_DAILY_FILE` en `.env`.
- **Nombre del archivo histórico:** el nombre predeterminado es `online_retail_II.csv`. Si el archivo disponible tiene
  otro nombre, se configura en `DATAMART_HISTORY_FILE` (`.env`) sin tocar el código.
  *(Anotar aquí cuál se usó y cuánto se solapa con `data.csv`.)*
- Las columnas se leen por nombre y no por posición, y todo se lee como texto: así un valor sucio no rompe la lectura,
  se convierte después y, si falla, se rechaza con su motivo.

## 8. Pendiente / cómo lo habría resuelto

- Descarga automática de los datasets desde Kaggle dentro del DAG (requiere credenciales de Kaggle como Variables/Secrets).
- Carga incremental particionada por fecha (sección 6).
- Afinar la heurística de categorías con el negocio.
