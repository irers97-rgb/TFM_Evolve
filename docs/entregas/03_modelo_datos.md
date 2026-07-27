# Entrega 3 - Diseño del modelo de datos y capa gold del proyecto

> **Nota de revisión (post feedback Julio Valero, Entrega 3):** este documento se ha revisado tras validar la periodicidad real de las fuentes públicas (EMH y ESCRI/SIAE). El proyecto reduce su alcance a un único MVP —el Módulo 1, predicción de demanda asistencial— apoyado en datos reales, ajustando la granularidad y el horizonte a lo que las fuentes realmente permiten. Los módulos 2 y 3 quedan documentados como arquitectura prevista pero **no implementada** en el MVP, dado que dependen de granularidades (ocupación diaria, episodios individuales, biomarcadores, reingreso a 30 días) que no están garantizadas por las fuentes públicas disponibles.

---

## 1. Resumen de la idea y datos del proyecto

### Problema que resuelve

Los reingresos evitables y las altas prematuras son uno de los problemas más costosos del sistema sanitario español, y la presión asistencial no anticipada agrava ambos. Cuando un hospital no puede prever con antelación un pico de demanda, la gestión de camas y personal se vuelve reactiva, lo que incrementa el riesgo de altas prematuras y de derivaciones de urgencia. Este proyecto se centra en la parte del problema que **puede sostenerse con evidencia real**: anticipar la demanda asistencial con la suficiente antelación para permitir una planificación de recursos menos reactiva.

### Solución que se quiere construir (MVP)

Un sistema de **predicción de demanda asistencial**: un modelo de forecasting que proyecta el volumen semanal de ingresos hospitalarios (urgentes y programados) por servicio y comunidad autónoma, con un horizonte de **4 a 8 semanas**, apoyado en la serie histórica real de la Encuesta de Morbilidad Hospitalaria (EMH).

**Fuera de alcance del MVP (arquitectura prevista, no implementada):**
- *Derivación inteligente entre centros:* requeriría ocupación hospitalaria con granularidad diaria, que ESCRI/SIAE no ofrece (es una foto anual, no una serie temporal de ocupación).
- *Scoring de riesgo de reingreso a 30 días:* requeriría episodios individuales, biomarcadores y una variable de reingreso real, ninguno disponible en fuentes públicas a nivel de paciente. Solo sería abordable con un dataset 100% sintético, lo que — como señaló el feedback recibido — permitiría demostrar un pipeline pero no sostener conclusiones clínicas.

Ambos módulos se mantienen documentados en la sección 10 como extensión futura, con los requisitos de datos que harían falta para implementarlos con garantías.

### Fuentes de datos y tipo de información aportada

| Fuente | Tipo de información | Uso en el MVP |
|---|---|---|
| Encuesta de Morbilidad Hospitalaria (EMH) – INE, serie 2014-2023 | Altas hospitalarias con fecha real de alta, diagnóstico CIE-10, modalidad de ingreso (urgente/programado), edad, sexo, CCAA | **Fuente principal del MVP.** Permite reconstruir una serie semanal real de ingresos a partir de la fecha de alta del microdato. |
| Estadística de Centros de Atención Especializada (antigua ESCRI) – Ministerio de Sanidad, serie 2014-2023 | Foto anual (a 31 de diciembre) de camas instaladas/en funcionamiento, dotación tecnológica y personal por centro | **Variable de contexto anual estática** (capacidad estructural), no como serie temporal de ocupación. |
| Grupos Relacionados por el Diagnóstico (GRD) – SNS | Estancia media y coste de referencia por diagnóstico | No se usa en el MVP (era soporte de los módulos 2 y 3, ambos fuera de alcance) |

---

## 2. Validación de las fuentes: qué granularidad soportan realmente

Este apartado responde directamente al comentario recibido: *"valida exactamente qué campos y frecuencias existen en EMH y ESCRI"*.

| Fuente | Periodicidad de publicación | Periodicidad real del dato | Consecuencia para el proyecto |
|---|---|---|---|
| **EMH (INE)** | Anual (un fichero de microdatos por año, publicado con 12-18 meses de retraso) | El microdato individual trae la **fecha real del alta**, no solo el año | Se puede reconstruir una serie **semanal real** agregando por fecha de alta dentro de cada fichero anual. No es un dato "en vivo": llega una vez al año, con retraso. |
| **ESCRI / SIAE (Sanidad)** | Anual | Foto única "a 31 de diciembre" de cada año — **no** es una serie de ocupación a lo largo del año | No permite construir `ocupacion_pct` semanal real. Solo aporta capacidad estructural (camas, dotación) como variable de contexto que cambia, como mucho, una vez al año. |

**Consecuencia directa sobre las promesas del producto:**
- El horizonte de predicción pasa de "24-72h en producción" a **"proyección semanal, 4-8 semanas vista, sobre datos históricos"**. No es un sistema de alerta en tiempo real (ninguna de las fuentes lo permite, ni siquiera en producción real, por el propio retraso de publicación de la EMH), sino una **herramienta de planificación estructural de temporada** (ej. anticipar el pico de invierno con semanas de antelación usando el patrón histórico).
- Las variables de ocupación dejan de tratarse como serie semanal y pasan a ser contexto anual estático.

---

## 3. Tecnología o formato de almacenamiento elegido

Se mantiene la combinación de **Parquet** para la capa `processed` (compresión columnar, lectura selectiva, compatibilidad con pandas/scikit-learn) y **CSV** para `raw` (formato nativo de las fuentes) y `gold` (facilita inspección manual y carga en herramientas de visualización tipo Power BI/Tableau).

No se utiliza base de datos relacional: el volumen no lo requiere, las fuentes son ficheros descargables sin actualización en streaming, y el proyecto no necesita transaccionalidad ni consultas concurrentes.

---

## 4. Estructura de capas de datos

```
data/
├── raw/          # Datos originales tal como se obtienen de la fuente
├── processed/    # Datos limpios, tipados y parcialmente transformados
└── gold/         # Dataset final listo para modelo, EDA y dashboard
```

| Capa | Contenido esperado | Granularidad |
|---|---|---|
| `raw/` | Ficheros de microdatos EMH descargados por año (uno por año, 2014-2023) y ficheros anuales de la Estadística de Centros de Atención Especializada, sin modificaciones. | Una entrega anual por fuente, con fecha de alta real en el caso de EMH. |
| `processed/` | Ficheros Parquet con datos limpios: tipos corregidos, nulos tratados, duplicados eliminados, columnas en `snake_case`, fechas en ISO 8601. Los registros individuales de EMH se agregan aquí a nivel semanal. | Una fila por alta individual (antes de agregar) → agregación a semana × servicio × CCAA. |
| `gold/` | Un único dataset final, contrato de datos del modelo, el EDA y el dashboard. | Semana epidemiológica × servicio × CCAA. |

---

## 5. Definición de la capa gold

### `gold_demanda_asistencial.csv`

**Descripción funcional:** Serie temporal semanal de ingresos hospitalarios por servicio y CCAA, reconstruida a partir de la fecha real de alta de los microdatos de la EMH, enriquecida con variables estacionales y con capacidad estructural anual (ESCRI) como contexto. Es la entrada única del modelo de forecasting del MVP.

**Granularidad:** una fila por **semana epidemiológica × servicio hospitalario × comunidad autónoma**.

**Número aproximado de registros:** ~50.000 filas (52 semanas × 10 años × ~10 servicios × 10 CCAA representativas).

**Campos principales:**

| Campo | Tipo | Descripción | Fuente real | Variable objetivo |
|---|---|---|---|---|
| `anio` | int | Año del registro | EMH | |
| `semana_epidemiologica` | int | Semana del año (1-53), derivada de la fecha real de alta | EMH (derivado) | |
| `ccaa_codigo` | str | Código INE de comunidad autónoma | EMH | |
| `servicio` | str | Servicio hospitalario, mapeado desde diagnóstico CIE-10 | EMH (derivado) | |
| `ingresos_urgentes` | int | Nº de ingresos urgentes en la semana, contados sobre fecha real de alta | EMH | ✅ Variable objetivo |
| `ingresos_programados` | int | Nº de ingresos programados en la semana | EMH | |
| `ratio_urgentes_programados` | float | Derivada: `ingresos_urgentes / ingresos_programados` | Derivado de EMH | |
| `indicador_estacional` | int | 1 si la semana cae en otoño-invierno (semanas 40-12), 0 en otro caso | Derivado del calendario | |
| `festivo_semana` | int | Número de festivos nacionales en la semana | Calendario oficial | |
| `camas_totales_ccaa` | int | Capacidad estructural de la CCAA (contexto **anual**, no varía dentro del año) | Estadística de Centros (antigua ESCRI) | |
| `tendencia_ingresos_4s` | float | Variación de `ingresos_urgentes` respecto a las 4 semanas anteriores | Derivado de EMH | |

**Cambios respecto a la versión anterior de este documento:** se eliminan `ocupacion_pct` y `tendencia_ocupacion_7d` como variables semanales, porque ESCRI no ofrece ocupación con esa granularidad. `camas_totales_ccaa` sustituye a la ocupación como variable de contexto, y se marca explícitamente como **estática dentro de cada año**.

**Clave primaria:** `(anio, semana_epidemiologica, ccaa_codigo, servicio)`

**Uso posterior:** entrenamiento y validación del modelo de forecasting (baseline + SARIMA/Prophet). Dashboard de predicción de demanda.

---

## 6. Diccionario de datos

| Campo | Descripción | Tipo de dato | Fuente | Obligatorio | Observaciones |
|---|---|---|---|---|---|
| `semana_epidemiologica` | Semana del año, derivada de la fecha real de alta del microdato EMH | int (1-53) | EMH (derivado) | Sí | Semana 53 solo en años con esa configuración de calendario; tratar con cuidado en el cambio de año |
| `ingresos_urgentes` | Volumen de ingresos urgentes en la semana | int | EMH | Sí | Variable objetivo del modelo |
| `ingresos_programados` | Volumen de ingresos programados en la semana | int | EMH | Sí | |
| `ratio_urgentes_programados` | Cociente entre ingresos urgentes y programados | float | Derivado | No (derivada) | Indicador de presión asistencial relativa |
| `indicador_estacional` | Binaria: 1 si otoño-invierno (semanas 40-12) | int (0/1) | Derivado del calendario | No (derivada) | Captura estacionalidad respiratoria/cardiovascular |
| `festivo_semana` | Número de festivos nacionales en la semana | int | Calendario oficial | No | |
| `camas_totales_ccaa` | Capacidad estructural de camas de la CCAA | int | Estadística de Centros (ESCRI) | Sí | **Estática dentro del año**; no confundir con ocupación en tiempo real |
| `tendencia_ingresos_4s` | Variación de ingresos respecto a las 4 semanas previas | float | Derivado | No (derivada) | |
| `servicio` | Servicio hospitalario asignado por regla diagnóstico → servicio | str | EMH (derivado) | Sí | La EMH no desagrega por servicio de forma nativa; requiere mapeo documentado |

---

## 7. Problemas de calidad esperados

- **Mapeo diagnóstico → servicio:** la EMH no desagrega nativamente por servicio hospitalario. Se necesita una regla de mapeo CIE-10 → servicio, que puede generar ambigüedad en códigos no clínicos (ej. códigos Z).
- **Retraso de publicación:** la EMH se publica con 12-18 meses de retraso; el último año de la serie puede estar en edición provisional. Esto es una limitación estructural del producto, no un defecto de calidad a corregir, y se documenta como tal en el alcance del MVP.
- **Ruptura estructural 2020-2021:** la pandemia introduce un cambio de patrón que se excluirá del entrenamiento del modelo o se tratará como variable exógena, según lo que muestre el EDA.
- **Semana epidemiológica en el cambio de año:** la semana 1 puede empezar en diciembre del año anterior; se tratará con cuidado para no introducir discontinuidades artificiales en el modelo de series temporales.
- **Heterogeneidad entre CCAA:** posibles diferencias en criterios de codificación y reporte entre comunidades, que pueden requerir normalización adicional.

---

## 8. Decisiones de limpieza y transformación previstas

### Tratamiento de nulos
- Registros de la EMH sin diagnóstico CIE-10 válido se excluyen, ya que impiden el mapeo a servicio.
- Si una CCAA no tiene registros de capacidad (`camas_totales_ccaa`) para un año concreto, se imputa con el último valor disponible (forward-fill), documentando la incidencia.

### Eliminación de duplicados
- Deduplicación de altas individuales por combinación de identificador de alta (si el microdato lo incluye) o por combinación de fecha + centro + diagnóstico antes de agregar a nivel semanal.

### Normalización de formatos
- Fechas en formato ISO 8601. Semana epidemiológica como entero (1-53).
- Nombres de columnas en `snake_case`, sin tildes ni caracteres especiales.
- Códigos CIE-10 normalizados a formato estándar, sin espacios, en mayúsculas.

### Variables derivadas a construir

| Variable derivada | Cálculo |
|---|---|
| `ratio_urgentes_programados` | `ingresos_urgentes / ingresos_programados` |
| `indicador_estacional` | 1 si `semana_epidemiologica` ∈ [40, 53] ∪ [1, 12], 0 en otro caso |
| `tendencia_ingresos_4s` | Diferencia de `ingresos_urgentes` respecto a las 4 semanas anteriores |

### Datos que se descartarán
- Años 2020 y 2021: ruptura estructural por la pandemia, se documentará la exclusión y se evaluará en el EDA si aporta valor como variable exógena.
- CCAA con series incompletas o con menos de 5 años de histórico disponible: se excluyen del entrenamiento y se documentan como limitación de cobertura.

### Criterio de validez de un registro
Un registro semanal se considera válido si: tiene diagnóstico principal mapeable a servicio, la semana epidemiológica está bien formada, y no es un duplicado de la combinación `(anio, semana_epidemiologica, ccaa_codigo, servicio)`.

---

## 9. Riesgos del modelo de datos

### ¿Qué parte del modelo de datos está más clara?
La reconstrucción de la serie semanal de ingresos a partir de la fecha real de alta de la EMH. Es una transformación directa sobre un dato real y verificable, sin necesidad de generación sintética.

### ¿Qué parte genera más incertidumbre?
El mapeo diagnóstico → servicio, porque la EMH no lo ofrece de forma nativa y la regla de mapeo puede introducir sesgo o ambigüedad en determinados diagnósticos.

### ¿Qué ocurriría si la reconstrucción semanal no tiene suficiente calidad?
Se simplificaría el horizonte a granularidad mensual, que es más robusta frente a ruido de mapeo y de calendario, sacrificando resolución pero manteniendo la validez del dato.

### ¿Qué alternativa tendríais para simplificar el modelo si fuera necesario?
Reducir el modelo a nivel nacional (sin desagregar por CCAA ni servicio) si la desagregación introduce demasiado ruido o huecos en la serie, priorizando tener una serie robusta sobre tener una serie muy desagregada pero poco fiable.

---

## 10. Extensión futura (fuera del MVP): Módulos 2 y 3

Se documentan aquí como arquitectura prevista, **no implementada**, junto con los requisitos de datos que harían falta para abordarlos con garantías, en línea con el feedback recibido.

| Módulo | Qué requeriría para ser viable con datos reales |
|---|---|
| **Derivación inteligente entre centros** | Una fuente de ocupación hospitalaria con granularidad diaria (no anual), actualmente no disponible en fuentes públicas abiertas en España. Sería viable con acceso a un sistema de información hospitalaria en tiempo real, fuera del alcance de un proyecto académico con datos públicos. |
| **Scoring de riesgo de reingreso a 30 días** | Episodios individuales con biomarcadores y variable de reingreso verificada, no disponibles en fuentes públicas a nivel de paciente. Solo sería abordable con un dataset sintético, que —tal como señaló el feedback— demuestra el pipeline técnico pero no sostiene conclusiones clínicas ni evidencia de capacidad asistencial real. |

Ambos módulos quedan fuera del MVP para evitar prometer una capacidad de decisión clínica o de derivación en tiempo real que los datos disponibles no pueden sostener.
