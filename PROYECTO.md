# Proyecto KDD — Análisis de Datos COVID-19 México

## Descripción general

Caso de estudio de Knowledge Discovery in Databases (KDD) sobre los registros abiertos de COVID-19 publicados por la Secretaría de Salud de México (SSa). El objetivo es construir un data warehouse en MongoDB, limpiar y unificar los datos, y realizar análisis sobre la evolución de la pandemia en México de 2020 a 2026.

**Fuente oficial de datos:**
[https://www.gob.mx/salud/documentos/datos-abiertos-152127](https://www.gob.mx/salud/documentos/datos-abiertos-152127)

**Entregable final requerido:**
- Esquema de data warehouse
- Tipos y fuentes de datos
- Técnicas de limpieza de datos
- Parámetros de configuración del data warehouse
- Repositorio con el conjunto de datos preprocesados

---

## Stack tecnológico

| Componente | Tecnología |
|---|---|
| Lenguaje | Python 3.12 |
| Gestión de entorno | `uv` |
| Base de datos | MongoDB (Docker) |
| Cliente Python | `pymongo 4.x` |
| Procesamiento | `pandas`, `polars` |
| Notebook | Jupyter (`main.ipynb`) |

---

## Infraestructura — MongoDB

- **URI:** `mongodb://admin:***@127.0.0.1:27017/?authSource=admin`
- **Base de datos:** `covid`
- **Credenciales:** cargadas desde `.env` con `python-dotenv`
- **Verificación:** ping al servidor retorna `{'ok': 1.0}` ✓

---

## Fuentes de datos

Archivos ZIP descargados manualmente en la carpeta `data/`:

| Archivo ZIP | CSV interno | Tamaño ZIP |
|---|---|---|
| `DATA_RAW_2020.zip` | `COVID19MEXICO2020.csv` | 59 MB |
| `DATA_RAW_2021.zip` | `COVID19MEXICO2021.csv` | 132 MB |
| `DATA_RAW_2022.zip` | `COVID19MEXICO2022.csv` | 93 MB |
| `DATA_RAW_2023.zip` | `COVID19MEXICO.csv` | 18 MB |
| `DATA_RAW_2024.zip` | `COVID19MEXICO.csv` | 2.8 MB |
| `DATA_RAW_2025.zip` | `COVID19MEXICO.csv` | 2.5 MB |
| `DATA_RAW_2026.zip` | `COVID19MEXICO.csv` | 3.7 MB |

**Nota:** Los archivos de 2023 en adelante usan el nombre genérico `COVID19MEXICO.csv` y tienen un esquema de columnas ligeramente diferente al de 2020–2022 (e.g. `CLASIFICACION_FINAL_COVID`, `CLASIFICACION_FINAL_FLU`, `RESULTADO_PCR` en lugar de `RESULTADO_LAB`).

### Esquema de columnas (2020–2022)

40 columnas: `FECHA_ACTUALIZACION`, `ID_REGISTRO`, `ORIGEN`, `SECTOR`, `ENTIDAD_UM`, `SEXO`, `ENTIDAD_NAC`, `ENTIDAD_RES`, `MUNICIPIO_RES`, `TIPO_PACIENTE`, `FECHA_INGRESO`, `FECHA_SINTOMAS`, `FECHA_DEF`, `INTUBADO`, `NEUMONIA`, `EDAD`, `NACIONALIDAD`, `EMBARAZO`, `HABLA_LENGUA_INDIG`, `INDIGENA`, `DIABETES`, `EPOC`, `ASMA`, `INMUSUPR`, `HIPERTENSION`, `OTRA_COM`, `CARDIOVASCULAR`, `OBESIDAD`, `RENAL_CRONICA`, `TABAQUISMO`, `OTRO_CASO`, `TOMA_MUESTRA_LAB`, `RESULTADO_LAB`, `TOMA_MUESTRA_ANTIGENO`, `RESULTADO_ANTIGENO`, `CLASIFICACION_FINAL`, `MIGRANTE`, `PAIS_NACIONALIDAD`, `PAIS_ORIGEN`, `UCI`

---

## Paso 1 — Carga de datos raw por año

### Estrategia

- Se crea una colección por año: `2020_raw`, `2021_raw`, …, `2026_raw`.
- Los CSV se leen directamente del ZIP (sin descomprimir a disco) en **chunks de 50,000 filas** para mantener el uso de RAM por debajo de ~200 MB por lote.
- Se crea un **índice único sobre `ID_REGISTRO`** en cada colección para detección de duplicados intra-colección.
- La inserción usa `ordered=False`: si un lote contiene duplicados, MongoDB los rechaza individualmente pero continúa con el resto.
- Los valores `NaN` de pandas se convierten a `None` (→ `null` en MongoDB) antes de insertar.

### Resultados de carga

| Colección | Documentos insertados |
|---|---|
| `2020_raw` | 3,868,396 |
| `2021_raw` | 8,830,345 |
| `2022_raw` | 6,451,944 |
| `2023_raw` | 1,222,219 |
| `2024_raw` | 177,618 |
| `2025_raw` | 157,142 |
| `2026_raw` | 217,324 |
| **TOTAL** | **20,924,988** |

### Nota sobre duplicados inter-colección

El índice único opera **por colección**, no de forma global. El paso siguiente (colección unificada) es el encargado de eliminar duplicados entre años. El total definitivo sin duplicados cross-colección es **20,768,066** registros.

---

## Paso 2 — Colección unificada (`covid_unified`) ✓

### Cambio oficial de esquema (8 de julio de 2024)

A partir del paquete del 8-jul-2024, la DGE amplió el sistema para registrar todos los virus respiratorios del SISVER, no solo COVID-19. Los cambios de variables fueron:

| Acción | Variable anterior | Variable nueva |
|---|---|---|
| Reemplazada | `RESULTADO_LAB` | `RESULTADO_PCR` + `RESULTADO_PCR_COINFECCION` |
| Renombrada | `CLASIFICACION_FINAL` | `CLASIFICACION_FINAL_COVID` |
| Agregada | — | `CLASIFICACION_FINAL_FLU` |

El corte en las colecciones raw quedó:
- **Esquema viejo:** `2020_raw`, `2021_raw`, `2022_raw`, `2023_raw`
- **Esquema nuevo:** `2024_raw`, `2025_raw`, `2026_raw`

### Estrategia de unificación

- **Esquema canónico:** se adopta el esquema nuevo (2024+).
- **Normalización 2020–2023:** `RESULTADO_LAB` → `RESULTADO_PCR`, `CLASIFICACION_FINAL` → `CLASIFICACION_FINAL_COVID`, campos nuevos (`RESULTADO_PCR_COINFECCION`, `CLASIFICACION_FINAL_FLU`) se agregan como `None`.
- **Deduplicación global:** índice único sobre `ID_REGISTRO` en `covid_unified`. Se procesa 2020 → 2026; en caso de ID duplicado entre años, gana el primer registro visto.
- **Eficiencia:** cursores MongoDB con `batch_size=50,000`.

### Resultados

| Año | Insertados | Duplicados descartados |
|---|---|---|
| 2020 | 3,868,396 | 0 |
| 2021 | 8,830,344 | 1 |
| 2022 | 6,451,941 | 3 |
| 2023 | 1,222,219 | 0 |
| 2024 | 177,618 | 0 |
| 2025 | 157,142 | 0 |
| 2026 | 60,406 | 156,918 |
| **TOTAL** | **20,768,066** | **156,922** |

Los 156,918 duplicados de 2026 corresponden a registros que ya existían en colecciones anteriores (principalmente IDs de 2025 incluidos en el ZIP de 2026).

---

## Paso 3 — Colección Durango (`durango`) ✓

**Criterio de filtro:** `ENTIDAD_RES = 10` OR `ENTIDAD_NAC = 10` (clave 10 = Durango en el catálogo SSa).

- Fuente: `covid_unified`
- Colección resultante: `durango`
- Documentos: **250,008**
- Índice único en `ID_REGISTRO`, inserción en lotes de 50K

---

## Paso 4 — Estadística descriptiva (Durango) ✓

Se analizan las variables **EDAD** e **HIPERTENSIÓN** de la colección `durango` con pandas y matplotlib.

### Catálogo oficial HIPERTENSIÓN (SSa)

| Valor | Significado |
|---|---|
| `1` | Sí |
| `2` | No |
| `97` | No aplica |
| `98` | Se ignora |
| `99` | No especificado |

En Durango solo aparecen valores `1`, `2` y `98`. Para covarianza se recodifica a binario (`1`→`1`, `2`→`0`); el resto queda como `NaN` y se excluye automáticamente.

### Técnica de limpieza aplicada

- EDAD e HIPERTENSIÓN ya vienen como `int` desde MongoDB, sin conversión necesaria.
- Los 2,928 registros con `HIPERTENSION=98` se excluyen solo del cálculo de covarianza; para las estadísticas de EDAD se usan los 250,008 registros completos.

### Estadísticas calculadas

- **EDAD:** Mínimo, Máximo, Promedio, Varianza, Percentiles (P25, P50, P75, P90, P95), Cuartiles (Q1, Q2, Q3) e IQR.
- **Covarianza:** matriz Cov(EDAD, HIPERTENSIÓN\_BIN) sobre 247,080 registros válidos.

### Visualizaciones generadas (`estadisticas_durango.png`)

1. Histograma de EDAD con líneas de media y mediana
2. Box plot de EDAD
3. Barras de distribución de HIPERTENSIÓN (Sí / No / Ignorado)
4. Box plot de EDAD segmentado por grupo de HIPERTENSIÓN
5. Curva de percentiles de EDAD

---

## Paso 5 — Integración de datos: ENTIDAD_UM = 10 ✓

Se simula integración de datos desde una fuente adicional: registros donde la **unidad médica** pertenece a Durango (`ENTIDAD_UM = 10`), que no estuvieran ya en la colección por `ENTIDAD_RES` o `ENTIDAD_NAC`.

**Por qué:** ampliar la cobertura del análisis a pacientes atendidos en Durango aunque no residan ni hayan nacido ahí, simulando la integración real de fuentes heterogéneas.

**Filtro aplicado en `covid_unified`:**
```
ENTIDAD_UM = 10  AND  ENTIDAD_RES ≠ 10  AND  ENTIDAD_NAC ≠ 10
```

Esta proyección exclusiva garantiza disjunción con los documentos existentes, por lo que el conteo de la consulta coincide exactamente con los insertados (0 duplicados).

- Registros integrados: **2,983**
- Total `durango` tras integración: **253,991**

---

## Paso 6 — Estadística descriptiva post-integración + comparativa ✓

Se repitió el análisis estadístico sobre la colección `durango` actualizada (253,991 registros) y se generó una tabla comparativa con columna Δ Diferencia para observar el impacto de los 2,983 registros de ENTIDAD_UM integrados.

Gráficas guardadas en `estadisticas_durango_actualizado.png`.

---

## Paso 7 — Enriquecimiento: campos de días calculados ✓

Se agregaron tres nuevos campos numéricos a cada documento de `durango`:

| Campo | Fórmula | Nulos |
|---|---|---|
| `DIAS_SINTOMAS` | `FECHA_INGRESO − FECHA_SINTOMAS` | Ninguno (todas las fechas válidas) |
| `DIAS_ATENCION` | `FECHA_DEF − FECHA_INGRESO` | 245,371 (no fallecidos) |
| `DIAS_ENFERMO` | `FECHA_DEF − FECHA_SINTOMAS` | 245,371 (no fallecidos) |

**Técnica de limpieza:** `FECHA_DEF = '9999-99-99'` se interpreta como ausencia de defunción y se convierte a `None`. Las fechas se parsean de `string` a `datetime` para la resta. Los campos se escriben de vuelta a MongoDB con `bulk_write` en lotes de 10,000.

- Fallecidos en Durango: **7,620** (campos completos)
- No fallecidos: **245,371** (DIAS_ATENCION y DIAS_ENFERMO = null)

---

## Paso 8 — Defunciones por residencia en Durango ✓

Conteo de registros en `durango` donde `ENTIDAD_RES = 10` y `FECHA_DEF ≠ '9999-99-99'`.

- **Defunciones de residentes en Durango: 4,909**

---

## Paso 9 — Corrección de defunciones perdidas por deduplicación ✓

**Problema detectado:** la estrategia "gana el primero" de `covid_unified` conservó versiones antiguas de 5 registros donde el paciente aún estaba vivo (`FECHA_DEF = '9999-99-99'`). En datasets posteriores esos mismos pacientes ya aparecían con fecha de defunción real, pero esa versión fue descartada como duplicado.

**Solución:**
1. Se recolectaron todos los IDs de fallecidos con `ENTIDAD_RES=10` de las colecciones raw.
2. Se cruzaron contra `covid_unified` para encontrar los 5 IDs afectados.
3. Se recuperó la `FECHA_DEF` real desde la raw correspondiente.
4. Se actualizaron `FECHA_DEF`, `DIAS_ATENCION` y `DIAS_ENFERMO` en `covid_unified` y `durango` con `bulk_write`.

**Resultado:** defunciones `ENTIDAD_RES=10` corregidas de **4,909 → 4,914**.

---

## Paso 10 — Tasa de mortalidad COVID-19 en Durango ✓

Se integra población del **Censo de Población y Vivienda 2020 (INEGI)** como segunda fuente de datos.

| Grupo | Población | Defunciones | Tasa (%) |
|---|---|---|---|
| Total | 1,832,650 | 4,914 | — |
| Hombres | 904,866 | 2,059 | — |
| Mujeres | 927,784 | 2,855 | — |

Fórmula: `(Defunciones / Población) × 100`, resultado a 4 decimales.

Gráficas: barras de tasa por grupo + comparativa distribución población vs defunciones. Guardado en `tasa_mortalidad_durango.png`.

---

## Paso 11 — Mapeo de catálogos y exportación ✓

### Campos eliminados de `durango`
- `ID_REGISTRO` — identificador interno sin valor epidemiológico
- `FECHA_ACTUALIZACION` — fecha de corte del dataset, no del evento clínico
- Se eliminó también el índice único sobre `ID_REGISTRO`

### Catálogos aplicados (fuente: `240708 Catalogos.xlsx`)
ORIGEN, SECTOR, SEXO, TIPO_PACIENTE, NACIONALIDAD, ENTIDAD_UM, ENTIDAD_NAC, ENTIDAD_RES, RESULTADO_ANTIGENO, RESULTADO_PCR, RESULTADO_PCR_COINFECCION, CLASIFICACION_FINAL_COVID, CLASIFICACION_FINAL_FLU, y 20 columnas SI/NO (DIABETES, HIPERTENSION, ASMA, etc.)

**Nota importante:** `MUNICIPIO` se crea con clave compuesta `ENTIDAD_RES_MUNICIPIO_RES` **antes** de mapear `ENTIDAD_RES`, ya que el mapeo convierte el valor numérico a nombre de estado. Luego se elimina `MUNICIPIO_RES`.

### Exportaciones CSV (en `data/`)
- `durango_sin_mapear.csv` — datos crudos sin catálogos
- `durango_mapeado.csv` — datos con todas las descripciones textuales

### Mapeo adicional aplicado
- `FECHA_DEF = '9999-99-99'` → `'NO FALLECIDO'` — el código oficial SSa para pacientes sin defunción registrada se convierte a descripción legible en la versión mapeada. El CSV sin mapear conserva el código original.
- `MUNICIPIO_RES` (código numérico) → `MUNICIPIO` (nombre del municipio). Durango tiene 39 municipios; el mapeo se aplicó al 100% de las 252,991 filas.

### Colección MongoDB
- `durango_mapeado` — 252,991 documentos con valores legibles

### Archivos CSV generados en `data/`
- `durango_sin_mapear.csv` — valores numéricos originales, incluye `9999-99-99`
- `durango_mapeado.csv` — todas las claves sustituidas por descripciones textuales

---

## Paso 12 — Reproducibilidad: notebook configurable por estado ✓

### Objetivo

Hacer el notebook completamente reproducible para cualquier entidad federativa de México sin modificar código. Basta con cambiar el archivo `.env`.

### Variables de configuración (`.env` / `.env.example`)

| Variable | Descripción | Ejemplo (Durango) |
|---|---|---|
| `ENTIDAD_CLAVE` | Clave numérica SSa/INEGI del estado | `10` |
| `ESTADO_NOMBRE` | Nombre en minúsculas para colecciones y CSV | `durango` |
| `ESTADO_POB_TOTAL` | Población total — Censo INEGI 2020 | `1832650` |
| `ESTADO_POB_HOMBRES` | Población masculina — Censo INEGI 2020 | `904866` |
| `ESTADO_POB_MUJERES` | Población femenina — Censo INEGI 2020 | `927784` |

### Variables derivadas generadas automáticamente

```python
COL_ESTADO     = ESTADO_NOMBRE                     # colección estado
COL_ESTADO_MAP = f"{ESTADO_NOMBRE}_mapeado"        # colección mapeada
CSV_SIN_MAPEAR = f"data/{ESTADO_NOMBRE}_sin_mapear.csv"
CSV_MAPEADO    = f"data/{ESTADO_NOMBRE}_mapeado.csv"
```

### Cambios aplicados al notebook

- Se insertaron 2 celdas de configuración justo después de la conexión a MongoDB: una markdown con la tabla de variables y una de código que lee el `.env`.
- 12 celdas actualizadas: todas las referencias a `10` (clave), `"durango"`, `db["durango"]`, `db["durango_mapeado"]`, rutas CSV y diccionario de población fueron reemplazadas por las variables de entorno.

### Para analizar otro estado

1. Copiar `.env.example` a `.env`
2. Ajustar las 5 variables
3. Ejecutar el notebook completo — sin tocar una sola línea de código

### Rama git

`feature/configurable-estado` — commit `5443164`
