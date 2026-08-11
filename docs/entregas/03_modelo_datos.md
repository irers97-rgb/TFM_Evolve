# Entrega 3 - Diseño del modelo de datos y capa gold del proyecto

> **Nota de revisión (continuación hacia TFM completo):** este documento recupera los tres módulos planteados originalmente en la Entrega 2 (predicción de demanda, derivación y scoring de alta), pero los reformula **a la granularidad que las fuentes públicas reales permiten sostener**, en lugar de la granularidad individual/tiempo real que exigiría un dataset sintético. La diferencia respecto a la versión anterior de este documento es metodológica, no de ambición: los tres módulos se mantienen, pero el Módulo 2 (derivación) y el Módulo 3 (scoring de alta) pasan de operar a nivel de paciente individual a operar a nivel de **patología × CCAA × año**, apoyados en fuentes reales ya validadas (EMH, ESCRI, GRD e iCMBD). No se genera ningún dataset sintético en ningún módulo.

---

## 1. Resumen de la idea y datos del proyecto

### Problema que resuelve

Los reingresos evitables, las altas prematuras y las derivaciones reactivas entre centros son problemas costosos y estructurales del sistema sanitario español, agravados por la presión asistencial no anticipada. Este proyecto aborda las tres caras del problema —cuánta demanda va a llegar, a qué centro debería derivarse un paciente cuando la red está saturada, y qué patologías/perfiles concentran el riesgo de reingreso— **con evidencia real y agregada**, sin recurrir a datos individuales simulados.

### Solución que se quiere construir (TFM completo)

Un sistema de apoyo a la decisión hospitalaria con tres módulos, todos construidos sobre la misma arquitectura de datos y las mismas fuentes públicas reales:

| Módulo | Qué resuelve | Granularidad | Fuente principal |
|---|---|---|---|
| **1. Predicción de demanda asistencial** | Proyección semanal de ingresos urgentes/programados, 4-8 semanas vista, por servicio y CCAA | Semana × servicio × CCAA | EMH, ESCRI |
| **2. Índice de priorización de derivación** | Identifica qué combinaciones patología × CCAA concentran mayor presión estructural de derivación, para apoyar decisiones de planificación de red (no la derivación individual de un paciente en tiempo real) | Patología (CIE-10 2 dígitos) × CCAA × año | EMH, ESCRI, GRD |
| **3. Índice de riesgo de reingreso al alta** | Estratifica qué combinaciones patología × edad × CCAA concentran mayor riesgo de reingreso a 30 días, para apoyar la revisión de protocolos de alta en esos segmentos | Patología × grupo de edad × CCAA × año | EMH, GRD, **iCMBD** (indicador oficial de tasa de reingresos) |

**Lo que cambia respecto al planteamiento original de la Entrega 2:** los módulos 2 y 3 no producen una recomendación para un paciente concreto en un momento concreto (eso exigiría ocupación de camas en tiempo real y trazabilidad individual entre ingresos, que ninguna fuente pública ofrece). Producen, en cambio, un **índice de riesgo/priorización por segmento** (patología × CCAA × edad), pensado para que dirección médica y gestión de camas identifiquen dónde reforzar protocolos o capacidad, de forma anticipada y basada en el patrón histórico real. Es un cambio de escala de decisión (de "este paciente, ahora" a "este segmento, esta temporada"), no una renuncia al alcance funcional.

### Fuentes de datos y tipo de información aportada

| Fuente | Tipo de información | Uso en el proyecto |
|---|---|---|
| Encuesta de Morbilidad Hospitalaria (EMH) – INE, serie 2014-2023 | Altas con fecha real de alta, diagnóstico CIE-10, modalidad de ingreso, edad, sexo, CCAA, traslados entre centros | Fuente principal de los tres módulos: serie semanal real (M1), tasas de traslado por patología/CCAA (M2), volumen de altas y estancia por segmento (M3) |
| Estadística de Centros de Atención Especializada (antigua ESCRI) – Ministerio de Sanidad, serie 2014-2023 | Foto anual de camas instaladas/en funcionamiento, dotación tecnológica y personal por centro | Contexto estructural anual estático en M1 y M2 (capacidad de la red) |
| Grupos Relacionados por el Diagnóstico (GRD) – SNS | Estancia media y coste de referencia por diagnóstico | Variable de desviación estancia real vs. esperada, usada en M2 (severidad) y M3 (proxy de complejidad clínica) |
| **iCMBD (indicadores y ejes de análisis del CMBD) – Ministerio de Sanidad** | Indicadores oficiales agregados calculados por el Ministerio sobre el CMBD: **tasa de reingresos**, tasa de mortalidad, estancia media, hospitalizaciones potencialmente evitables, por CCAA/hospital/diagnóstico | **Fuente que hace viable el Módulo 3 sin datos sintéticos.** El Ministerio calcula el reingreso a partir de la trazabilidad real del paciente en el CMBD (a la que el público no tiene acceso) y publica el resultado agregado, que sí es de acceso público. |

---

## 2. Validación de las fuentes: qué granularidad soportan realmente

| Fuente | Periodicidad de publicación | Periodicidad real del dato | Consecuencia para el proyecto |
|---|---|---|---|
| **EMH (INE)** | Anual (12-18 meses de retraso) | Microdato con fecha real de alta | Serie semanal real para M1; agregación por patología/CCAA/año para M2 y M3 |
| **ESCRI / SIAE (Sanidad)** | Anual | Foto a 31 de diciembre, no serie de ocupación | Contexto estructural anual estático (M1, M2), no ocupación en tiempo real |
| **GRD (SNS)** | Anual | Estancia media y coste esperado por diagnóstico | Variable de desviación de estancia (M2, M3) |
| **iCMBD (Sanidad)** | Indicador consultable, actualización periódica según convenio Ministerio–Universidad de Cantabria | Indicador ya agregado por el Ministerio (no microdato) | Aporta directamente `tasa_reingreso` como variable objetivo real de M3, sin necesidad de reconstruirla desde microdatos individuales |

**Punto de atención abierto (a validar al inicio del TFM):** la página de microdatos de la EMH indica que, aunque los resultados agregados son de descarga libre, los **ficheros de microdatos individuales solo se facilitan bajo condiciones especiales** en el Área de Atención a Usuarios del INE, no por descarga directa sin trámite. Esto no invalida el proyecto (los datos agregados por patología/CCAA/año, que es la granularidad que usan M2 y M3, sí son de descarga abierta), pero si el Módulo 1 requiere el microdato con fecha exacta de alta para reconstruir la serie semanal, es necesario solicitar el fichero por ese canal y documentar el plazo de respuesta como riesgo de calendario del proyecto. De igual modo, el acceso exacto (consulta web vs. descarga de tabla) y la granularidad real de iCMBD (¿llega a nivel de patología × CCAA × año, o solo a nivel de hospital/CCAA sin desagregar por diagnóstico?) debe confirmarse en la fase de obtención de datos, documentando el resultado como parte del diario de proyecto.

---

## 3. Tecnología o formato de almacenamiento elegido

Se mantiene **Parquet** para la capa `processed` y **CSV** para `raw` y `gold`. No se utiliza base de datos relacional: las tres fuentes son ficheros o consultas descargables sin actualización en streaming, y el volumen (miles de filas agregadas, no microdatos masivos) no exige transaccionalidad ni consultas concurrentes.

---

## 4. Estructura de capas de datos

```
data/
├── raw/          # Datos originales tal como se obtienen de cada fuente
├── processed/    # Datos limpios, tipados y parcialmente transformados
└── gold/         # Tres datasets finales, uno por módulo, listos para modelo y dashboard
```

| Capa | Contenido esperado | Granularidad |
|---|---|---|
| `raw/` | Microdatos EMH por año, ficheros anuales ESCRI, tablas GRD, extracciones de indicadores iCMBD (reingreso, mortalidad, estancia) | Una entrega anual por fuente |
| `processed/` | Ficheros Parquet limpios: tipos corregidos, nulos tratados, duplicados eliminados, `snake_case`, fechas ISO 8601 | EMH agregado a semana (M1) y a patología×CCAA×año (M2, M3) |
| `gold/` | Tres datasets finales, uno por módulo | Ver sección 5 |

---

## 5. Definición de la capa gold

### 5.1 `gold_demanda_asistencial.csv` (Módulo 1 — sin cambios respecto a la versión anterior)

**Granularidad:** semana epidemiológica × servicio × CCAA.
**Clave primaria:** `(anio, semana_epidemiologica, ccaa_codigo, servicio)`.
**Campos principales:** `ingresos_urgentes` (objetivo), `ingresos_programados`, `ratio_urgentes_programados`, `indicador_estacional`, `festivo_semana`, `camas_totales_ccaa`, `tendencia_ingresos_4s`.
**Uso posterior:** forecasting SARIMA/Prophet, dashboard de demanda.

### 5.2 `gold_presion_derivacion.csv` (Módulo 2 — nuevo)

**Descripción funcional:** para cada patología y CCAA, cuantifica cuánta presión estructural de derivación existe, combinando frecuencia real de traslados, severidad clínica y capacidad de la red. Es la entrada del modelo de priorización de derivación.

**Granularidad:** patología (CIE-10, 2 dígitos) × CCAA × año.
**Número aproximado de registros:** ~9 años × 17 CCAA × ~20-25 grupos diagnósticos ≈ 3.000-4.000 filas.

| Campo | Tipo | Descripción | Fuente |
|---|---|---|---|
| `anio` | int | Año del registro | EMH |
| `ccaa_codigo` | str | Código INE de CCAA | EMH |
| `patologia_cie10` | str | Grupo diagnóstico (CIE-10, 2 dígitos) | EMH |
| `altas_totales` | int | Nº de altas de esa patología/CCAA/año | EMH |
| `traslados_totales` | int | Nº de traslados a otro centro | EMH |
| `tasa_traslado` | float | `traslados_totales / altas_totales` | Derivado |
| `mortalidad_intrahosp_pct` | float | % de altas por fallecimiento | EMH |
| `estancia_media_dias` | float | Estancia media real | EMH |
| `estancia_media_grd_esperada` | float | Estancia esperada según GRD | GRD |
| `desviacion_estancia` | float | `estancia_media_dias - estancia_media_grd_esperada` | Derivado |
| `camas_totales_ccaa` | int | Capacidad estructural de la CCAA (contexto anual) | ESCRI |
| `score_presion_derivacion` | float | **Variable de salida**: índice combinado (ver Entrega 4) | Derivado (modelo) |

**Clave primaria:** `(anio, ccaa_codigo, patologia_cie10)`.
**Uso posterior:** modelo de priorización de derivación (Entrega 4), panel de mapa de presión por CCAA/patología (Entrega 5).

### 5.3 `gold_riesgo_reingreso.csv` (Módulo 3 — nuevo)

**Descripción funcional:** para cada patología, grupo de edad y CCAA, combina la tasa de reingreso real (iCMBD) con variables clínicas y operativas agregadas (EMH, GRD) que la explican. Es la entrada del modelo de riesgo de reingreso al alta.

**Granularidad:** patología (CIE-10, 2 dígitos) × grupo de edad (franjas decenales) × CCAA × año.
**Número aproximado de registros:** ~9 años × 17 CCAA × ~20 patologías × ~8 grupos de edad ≈ 20.000-25.000 filas (con huecos esperables en combinaciones de bajo volumen).

| Campo | Tipo | Descripción | Fuente |
|---|---|---|---|
| `anio` | int | Año del registro | EMH / iCMBD |
| `ccaa_codigo` | str | Código INE de CCAA | EMH |
| `patologia_cie10` | str | Grupo diagnóstico | EMH |
| `grupo_edad` | str | Franja decenal | EMH |
| `altas_totales` | int | Nº de altas del segmento | EMH |
| `tasa_reingreso_30d` | float | **Variable objetivo**: tasa oficial de reingreso a 30 días | iCMBD |
| `estancia_media_dias` | float | Estancia media real del segmento | EMH |
| `desviacion_estancia` | float | Estancia real vs. GRD esperada | GRD (derivado) |
| `mortalidad_intrahosp_pct` | float | % de altas por fallecimiento del segmento | EMH |
| `pct_ingreso_urgente` | float | % de ingresos de carácter urgente en el segmento | EMH |
| `camas_totales_ccaa` | int | Contexto estructural anual | ESCRI |
| `score_riesgo_reingreso` | float | **Variable de salida**: riesgo predicho/explicado por el modelo | Derivado (modelo) |

**Clave primaria:** `(anio, ccaa_codigo, patologia_cie10, grupo_edad)`.
**Uso posterior:** modelo explicativo de riesgo de reingreso (Entrega 4), panel de riesgo por segmento (Entrega 5).

**Nota importante sobre lo que este módulo NO hace:** no asigna un riesgo a un paciente concreto antes de su alta (eso exigiría datos individuales que no están disponibles públicamente). Asigna un riesgo al **segmento** al que ese paciente pertenece, como apoyo a la revisión de protocolos, no como una decisión clínica individual.

---

## 6. Diccionario de datos (campos nuevos, M2 y M3)

| Campo | Descripción | Tipo | Fuente | Obligatorio | Observaciones |
|---|---|---|---|---|---|
| `patologia_cie10` | Grupo diagnóstico a 2 dígitos CIE-10 | str | EMH | Sí | Mismo nivel de agregación en M2 y M3 para permitir cruces entre módulos |
| `tasa_traslado` | Traslados sobre altas totales | float | Derivado de EMH | No (derivada) | Proxy de presión de derivación por patología/CCAA |
| `tasa_reingreso_30d` | Tasa oficial de reingreso a 30 días | float | iCMBD | Sí | Variable objetivo de M3; validar granularidad exacta disponible (ver sección 2) |
| `desviacion_estancia` | Estancia real menos estancia esperada por GRD | float | Derivado | No (derivada) | Predictor de severidad/complejidad en M2 y M3 |
| `score_presion_derivacion` | Índice combinado de presión de derivación | float | Derivado (modelo) | Sí (salida) | Ver Entrega 4 para la fórmula/modelo |
| `score_riesgo_reingreso` | Índice combinado de riesgo de reingreso | float | Derivado (modelo) | Sí (salida) | Ver Entrega 4 para la fórmula/modelo |

---

## 7. Problemas de calidad esperados

- **Mapeo diagnóstico → servicio** (M1) y **agregación CIE-10 a 2 dígitos** (M2, M3): ambos requieren reglas de mapeo documentadas que pueden introducir ambigüedad en códigos no clínicos.
- **Acceso a microdatos EMH:** sujeto a solicitud especial (ver sección 2); riesgo de calendario si el trámite se demora.
- **Granularidad real de iCMBD:** a confirmar si el indicador de tasa de reingresos desagrega por patología y CCAA simultáneamente, o solo por uno de los dos ejes. Si solo desagrega por CCAA (sin patología), el Módulo 3 se simplifica a granularidad CCAA × edad × año, documentando la pérdida de resolución.
- **Combinaciones de bajo volumen:** patologías poco frecuentes en CCAA pequeñas pueden generar tasas inestables (pocos casos, tasa muy sensible); se aplicará un umbral mínimo de altas por celda para incluir un registro en el entrenamiento.
- **Ruptura estructural 2020-2021:** igual que en M1, se excluye o se trata como variable exógena.

---

## 8. Decisiones de limpieza y transformación previstas

### Tratamiento de nulos
- Igual que en la versión anterior para M1.
- En M2 y M3, celdas con `altas_totales` por debajo de un umbral mínimo (a definir en el EDA, orientativamente 30-50 altas/año) se excluyen del entrenamiento por inestabilidad estadística, documentando la exclusión.

### Variables derivadas a construir

| Variable derivada | Cálculo | Módulo |
|---|---|---|
| `tasa_traslado` | `traslados_totales / altas_totales` | M2 |
| `desviacion_estancia` | `estancia_media_dias - estancia_media_grd_esperada` | M2, M3 |
| `score_presion_derivacion` | Combinación ponderada/aprendida de `tasa_traslado`, `mortalidad_intrahosp_pct`, `desviacion_estancia`, capacidad relativa | M2 |
| `score_riesgo_reingreso` | Salida del modelo explicativo entrenado sobre `tasa_reingreso_30d` | M3 |

### Datos que se descartarán
- Años 2020-2021 (igual que M1).
- Combinaciones patología × CCAA (× edad en M3) con volumen insuficiente (ver sección 7).

---

## 9. Riesgos del modelo de datos

### ¿Qué parte está más clara?
La reconstrucción del Módulo 1 (ya validada) y la disponibilidad de `tasa_reingreso_30d` como indicador oficial ya calculado por el Ministerio, que evita la necesidad de reconstruir el reingreso desde microdatos individuales.

### ¿Qué parte genera más incertidumbre?
La granularidad real y el modo de acceso al indicador iCMBD: si solo permite consulta interactiva (no descarga masiva) o si su desagregación no llega al cruce patología × CCAA, condiciona directamente el diseño del Módulo 3.

### Alternativa si iCMBD no ofrece la granularidad necesaria
Simplificar el Módulo 3 a nivel CCAA × año (sin desagregar por patología ni edad), manteniendo la validez del dato aunque con menor resolución, siguiendo el mismo principio de "serie robusta sobre serie muy desagregada pero poco fiable" ya aplicado en M1.

### Alternativa si el acceso a microdatos EMH se demora
Construir primero M2 y M3 (que solo requieren datos agregados, de descarga abierta) y dejar M1 a la espera del trámite de microdatos, documentando la dependencia como riesgo de calendario del proyecto.

---

## 10. Módulos 2 y 3: por qué se descarta la versión en tiempo real / individual

Se documenta aquí, para trazabilidad, por qué el diseño converge en un índice agregado por segmento y no en una recomendación individual en tiempo real (que sí aparece en referencias de otros TFM del mismo dominio):

| Elemento que exigiría un sistema en tiempo real/individual | Por qué no es viable con fuentes públicas | Solución adoptada |
|---|---|---|
| Ocupación de camas por centro, actualizada varias veces al día | ESCRI es una foto anual, no una serie de ocupación | Capacidad estructural anual como contexto, no como variable de disponibilidad en tiempo real |
| Identificador de paciente para vincular ingreso y reingreso | La EMH tiene como unidad estadística el alta, no el paciente; no hay trazabilidad pública entre altas de la misma persona | Se usa `tasa_reingreso_30d` de iCMBD, ya calculada por el Ministerio con la trazabilidad interna que el público no tiene |
| Biomarcadores y constantes vitales individuales al momento del alta | No publicados a nivel de paciente en ninguna fuente pública | Se sustituyen por proxies agregados de severidad (mortalidad, desviación de estancia) a nivel de segmento |

Esta tabla sustituye a la sección de "extensión futura, no implementada" de la versión anterior: los módulos 2 y 3 **sí se implementan**, pero a la escala de decisión que los datos reales permiten sostener con garantías.
