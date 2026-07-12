# Entrega 3 - Diseño del modelo de datos y capa gold del proyecto

---

## 1. Resumen de la idea y datos del proyecto

### Problema que resuelve

Los reingresos evitables y las altas prematuras son uno de los problemas más costosos del sistema sanitario español. Cuando un hospital opera bajo presión asistencial elevada, la decisión de dar el alta a un paciente se ve condicionada por la necesidad de liberar camas, lo que puede derivar en reingresos con peor pronóstico clínico y mayor coste para el sistema. De forma paralela, la derivación de pacientes entre centros hospitalarios se realiza frecuentemente de manera reactiva, cuando el hospital ya ha alcanzado su límite de capacidad. Ambos problemas comparten una raíz común: la ausencia de herramientas basadas en datos que apoyen estas decisiones de forma objetiva, homogénea y anticipada.

### Solución que se quiere construir

Un sistema de apoyo a la decisión clínica compuesto por tres módulos interconectados:

- **Módulo 1 – Predicción de demanda asistencial:** forecasting del volumen de ingresos hospitalarios en horizonte de 1 a 7 días.
- **Módulo 2 – Derivación inteligente:** algoritmo de puntuación ponderada para orientar el traslado de pacientes al centro más adecuado de la red.
- **Módulo 3 – Scoring de alta:** modelo de clasificación supervisada que estima el riesgo de reingreso a 30 días para pacientes candidatos al alta.

### Fuentes de datos y tipo de información aportada

| Fuente | Tipo de información |
|---|---|
| Encuesta de Morbilidad Hospitalaria (EMH) – INE, serie 2014-2023 | Altas, diagnósticos CIE-10, estancias, reingresos, traslados, mortalidad por patología, edad, sexo y CCAA |
| Estadística de Establecimientos Sanitarios con Régimen de Internado (ESCRI) – Ministerio de Sanidad, serie 2014-2023 | Ocupación de camas, rotación, dotación tecnológica, presión de urgencias por centro |
| Grupos Relacionados por el Diagnóstico (GRD) – SNS | Coste medio por episodio y estancia media de referencia por diagnóstico |
| Dataset sintético generado ad hoc | Registros clínicos individuales simulados a partir de las distribuciones estadísticas públicas, usados para entrenamiento y validación del Módulo 3 |

---

## 2. Tecnología o formato de almacenamiento elegido

Se utilizará una combinación de **ficheros Parquet** como formato principal de almacenamiento estructurado y **ficheros CSV** para las capas de datos raw y para los outputs finales de la capa gold consumidos por herramientas de visualización.

### Justificación

- **Parquet** es el formato idóneo para la capa processed y para los datasets de entrenamiento de modelos. Ofrece compresión columnar eficiente, lectura selectiva de columnas y compatibilidad nativa con pandas y scikit-learn, lo que encaja perfectamente con el volumen de datos del proyecto (series de diez años con granularidad de episodio de alta o semana epidemiológica). En un proyecto académico sin infraestructura de Big Data, Parquet permite simular buenas prácticas de ingeniería de datos sin necesidad de un cluster.

- **CSV** se mantiene en la capa raw porque las fuentes públicas del INE y del Ministerio de Sanidad se distribuyen en ese formato, y en la capa gold porque facilita la inspección manual, la carga en herramientas de visualización como Power BI o Tableau y la entrega de artefactos reproducibles sin dependencias.

- No se utilizará base de datos relacional porque el volumen de datos no lo requiere, las fuentes son archivos descargables que no se actualizan en streaming en el contexto académico, y añadiría una capa de complejidad innecesaria. El proyecto no necesita transaccionalidad ni consultas concurrentes.

---

## 3. Estructura de capas de datos

```
data/
├── raw/          # Datos originales tal como se obtienen de la fuente
├── processed/    # Datos limpios, tipados y parcialmente transformados
└── gold/         # Datasets finales listos para modelos, análisis y dashboard
```

| Capa | Contenido esperado |
|---|---|
| `raw/` | Ficheros CSV descargados directamente del INE y del Ministerio de Sanidad, sin modificaciones. También el script de generación del dataset sintético y su output raw. Los ficheros se guardan con el nombre original de la fuente más la fecha de descarga. |
| `processed/` | Ficheros Parquet con datos limpios: tipos corregidos, nulos tratados, duplicados eliminados, columnas renombradas a convención snake_case, fechas normalizadas a ISO 8601. Una carpeta por fuente. |
| `gold/` | Ficheros CSV finales, uno por módulo funcional del sistema. Son el contrato de datos que consumirán los modelos, el EDA y el dashboard. |

---

## 4. Definición de la capa gold

La capa gold está compuesta por cuatro datasets, uno por módulo funcional más uno de referencia de costes compartido entre módulos.

---

### 4.1 `gold_demanda_asistencial.csv`

**Descripción funcional:** Serie temporal semanal de actividad hospitalaria por servicio y CCAA, enriquecida con variables estacionales y de contexto epidemiológico. Es la entrada principal del Módulo 1 de forecasting.

**Granularidad:** Una fila por semana epidemiológica × servicio hospitalario × comunidad autónoma.

**Número aproximado de registros:** ~50.000 filas (52 semanas × 10 años × ~10 servicios × 10 CCAA representativas).

**Campos principales:**

| Campo | Tipo | Descripción | Variable objetivo |
|---|---|---|---|
| `anio` | int | Año del registro | |
| `semana_epidemiologica` | int | Semana del año (1-53) | |
| `ccaa_codigo` | str | Código INE de comunidad autónoma | |
| `servicio` | str | Servicio hospitalario (medicina interna, cirugía, etc.) | |
| `ingresos_urgentes` | int | Número de ingresos urgentes en la semana | ✅ Variable objetivo M1 |
| `ingresos_programados` | int | Número de ingresos programados | |
| `ocupacion_pct` | float | Porcentaje de ocupación media semanal | |
| `estancia_media_dias` | float | Estancia media en días | |
| `indicador_estacional` | int | 1 si otoño/invierno, 0 si primavera/verano | |
| `festivo_semana` | int | Número de festivos nacionales en la semana | |
| `ratio_urgentes_programados` | float | Ratio derivada: ingresos urgentes / programados | |
| `tendencia_ocupacion_7d` | float | Variación de ocupación respecto a las 7 semanas anteriores | |

**Clave primaria:** `(anio, semana_epidemiologica, ccaa_codigo, servicio)`

**Uso posterior:** Entrenamiento y validación del Módulo 1 (SARIMA + Prophet + LSTM). Dashboard de predicción de demanda.

---

### 4.2 `gold_red_hospitalaria.csv`

**Descripción funcional:** Catálogo estático de hospitales de la red con sus capacidades y especialidades, más el estado de ocupación agregado. Es la entrada del Módulo 2 de derivación.

**Granularidad:** Una fila por hospital × año.

**Número aproximado de registros:** ~3.000 filas (≈300 hospitales SNS × 10 años).

**Campos principales:**

| Campo | Tipo | Descripción |
|---|---|---|
| `hospital_id` | str | Identificador único del centro (código ESCRI) |
| `anio` | int | Año del registro |
| `ccaa_codigo` | str | Comunidad autónoma |
| `tipo_centro` | str | Hospital general, especializado, comarcal |
| `camas_totales` | int | Camas en funcionamiento |
| `camas_uci` | int | Camas UCI disponibles |
| `ocupacion_media_anual_pct` | float | Ocupación media anual |
| `indice_rotacion` | float | Índice de rotación de camas |
| `presion_urgencias_pct` | float | Porcentaje de ingresos procedentes de urgencias |
| `dispone_uci` | int | Binaria: 1 si tiene UCI, 0 si no |
| `dispone_hemodinamica` | int | Binaria: capacidad de hemodinámica |
| `latitud` | float | Coordenada geográfica |
| `longitud` | float | Coordenada geográfica |

**Clave primaria:** `(hospital_id, anio)`

**Uso posterior:** Módulo 2 de derivación inteligente. Dashboard de red hospitalaria.

---

### 4.3 `gold_episodios_alta.csv`

**Descripción funcional:** Dataset de episodios de alta hospitalaria anonimizados (o sintéticos en el contexto del TFM), con variables clínicas, demográficas y de resultado. Es la entrada principal del Módulo 3 de scoring de alta.

**Granularidad:** Una fila por episodio de alta.

**Número aproximado de registros:** ~200.000 filas (dataset sintético calibrado sobre distribuciones EMH 2014-2023).

**Campos principales:**

| Campo | Tipo | Descripción | Variable objetivo |
|---|---|---|---|
| `episodio_id` | str | Identificador pseudonimizado del episodio | PK |
| `anio` | int | Año del alta | |
| `semana_epidemiologica` | int | Semana del alta | |
| `ccaa_codigo` | str | Comunidad autónoma | |
| `franja_edad` | str | Franja de 10 años (ej. "65-74") | |
| `sexo` | str | M / F | |
| `diagnostico_principal_cie10` | str | Código CIE-10 de diagnóstico principal | |
| `grp_grd` | str | Grupo relacionado por diagnóstico | |
| `tipo_ingreso` | str | Urgente / Programado | |
| `estancia_dias` | int | Días de hospitalización | |
| `estancia_media_grd` | float | Estancia media de referencia del GRD | |
| `ratio_estancia_grd` | float | Derivada: estancia_dias / estancia_media_grd | |
| `charlson_index` | int | Índice de comorbilidad de Charlson (0-37) | |
| `num_ingresos_previos_12m` | int | Número de ingresos en los 12 meses anteriores | |
| `biomarcador_troponina` | float | Valor de troponina (µg/L), -1 si no aplica | |
| `biomarcador_pct` | float | Procalcitonina (ng/mL), -1 si no aplica | |
| `biomarcador_bnp` | float | BNP (pg/mL), -1 si no aplica | |
| `pendencias_diagnosticas` | int | Binaria: 1 si hay pruebas pendientes al alta | |
| `constantes_estables_48h` | int | Binaria: 1 si constantes estables en últimas 48h | |
| `reingreso_30d` | int | **Variable objetivo**: 1 si reingresó en 30 días, 0 si no | ✅ |
| `traslado` | int | Binaria: 1 si el alta fue un traslado a otro centro | |
| `exitus` | int | Binaria: 1 si el paciente falleció durante el ingreso | |

**Clave primaria:** `episodio_id`

**Uso posterior:** Entrenamiento y validación del Módulo 3 (XGBoost + SHAP). EDA de perfil de riesgo. Dashboard de scoring.

---

### 4.4 `gold_referencia_grd.csv`

**Descripción funcional:** Tabla de referencia de costes y estancias medias por GRD. Dataset auxiliar compartido por los módulos 2 y 3 para calcular variables derivadas de coste y desviación de estancia.

**Granularidad:** Una fila por código GRD.

**Número aproximado de registros:** ~600 filas (catálogo GRD del SNS).

**Campos principales:**

| Campo | Tipo | Descripción |
|---|---|---|
| `grd_codigo` | str | Código del Grupo Relacionado por el Diagnóstico | PK |
| `grd_descripcion` | str | Descripción textual del GRD | |
| `estancia_media_dias` | float | Estancia media de referencia nacional | |
| `coste_medio_euros` | float | Coste medio por episodio según SNS | |
| `peso_relativo` | float | Peso relativo del GRD (complejidad) | |
| `especialidad` | str | Especialidad médica asociada | |

**Clave primaria:** `grd_codigo`

**Uso posterior:** Join con `gold_episodios_alta` para calcular `ratio_estancia_grd`. Análisis económico del dashboard.

---

### Resumen de la capa gold

| Dataset gold | Granularidad | Campos clave | Uso posterior |
|---|---|---|---|
| `gold_demanda_asistencial.csv` | Una fila por semana × servicio × CCAA | `semana_epidemiologica`, `ccaa_codigo`, `servicio`, `ingresos_urgentes` | Módulo 1 – forecasting, dashboard |
| `gold_red_hospitalaria.csv` | Una fila por hospital × año | `hospital_id`, `anio`, `ocupacion_media_anual_pct` | Módulo 2 – derivación, dashboard |
| `gold_episodios_alta.csv` | Una fila por episodio de alta | `episodio_id`, `charlson_index`, `reingreso_30d` | Módulo 3 – scoring, EDA |
| `gold_referencia_grd.csv` | Una fila por código GRD | `grd_codigo`, `estancia_media_dias`, `coste_medio_euros` | Join auxiliar M2 y M3 |

---

## 5. Relaciones entre datos

El proyecto trabaja con cuatro datasets en la capa gold que se relacionan de la siguiente forma:

```
gold_demanda_asistencial
  ccaa_codigo  ──────────────────────────────────────── (contexto CCAA)
                                                             │
gold_red_hospitalaria                                        │
  hospital_id (PK)                                          │
  ccaa_codigo  ─────────────────────────────── agrupa hospitales por CCAA

gold_episodios_alta
  episodio_id (PK)
  grp_grd  ──── N ─────── 1 ──── gold_referencia_grd.grd_codigo
  ccaa_codigo  ─────────────────── (contexto geográfico)
```

**Relaciones formales:**

- `gold_episodios_alta.grp_grd` → `gold_referencia_grd.grd_codigo`: relación **N:1**. Muchos episodios pertenecen al mismo GRD. Se usa para enriquecer cada episodio con el coste medio y la estancia de referencia mediante un left join.

- `gold_episodios_alta.ccaa_codigo` → `gold_demanda_asistencial.ccaa_codigo`: relación contextual **N:N** (no join directo). Se usa para cruzar el perfil de presión asistencial de la CCAA en el momento del alta con el riesgo de reingreso del episodio. Se materializa como variable derivada en la capa processed.

- `gold_red_hospitalaria` se consume de forma independiente en el Módulo 2. No hace join directo con los episodios de alta, ya que en el contexto del proyecto la derivación opera sobre el estado de la red en un instante dado, no sobre el histórico de episodios.

**Posibles problemas al combinar fuentes:**

- La EMH trabaja a nivel de CCAA y la ESCRI a nivel de centro hospitalario. El cruce geográfico requiere una tabla de correspondencia hospital → CCAA que debe construirse manualmente a partir del catálogo ESCRI.
- Los códigos GRD pueden variar entre versiones anuales del catálogo SNS. Se fijará la versión del año con mayor cobertura en la serie (GRD-SNS v30 o la más reciente disponible) y se anotará en el diccionario.
- El dataset sintético del Módulo 3 no tiene correspondencia directa con episodios reales de la EMH. Su uso es exclusivamente para entrenamiento y validación, no para análisis descriptivo sobre el sistema real.

---

## 6. Diccionario de datos inicial

A continuación se documenta el diccionario de los campos más relevantes para el análisis y el modelo. Se excluyen campos auxiliares obvios (año, CCAA).

| Campo | Descripción | Tipo de dato | Fuente | Obligatorio | Observaciones |
|---|---|---|---|---|---|
| `semana_epidemiologica` | Número de semana del año según calendario epidemiológico | int (1-53) | EMH / derivado de fecha | Sí | Semana 53 solo en años bisiesto; tratar con cuidado |
| `ingresos_urgentes` | Volumen de ingresos procedentes de urgencias en la semana | int | EMH / ESCRI | Sí | Variable objetivo M1 |
| `ocupacion_pct` | Porcentaje de ocupación de camas en el período | float (0-100) | ESCRI | Sí | Valores > 100 posibles en picos; no filtrar automáticamente |
| `presion_urgencias_pct` | Proporción de ingresos desde urgencias sobre total | float (0-100) | ESCRI | Sí | Indicador de saturación estructural |
| `indice_rotacion` | Número de pacientes por cama y año | float | ESCRI | Sí | |
| `diagnostico_principal_cie10` | Código CIE-10 del diagnóstico principal al alta | str | EMH / HIS sintético | Sí | Puede requerir agrupación por capítulo CIE para modelos |
| `charlson_index` | Índice de comorbilidad de Charlson (suma de pesos de comorbilidades) | int (0-37) | Derivado HIS / sintético | Sí | Variable predictora clave M3; valor 0 = sin comorbilidades |
| `num_ingresos_previos_12m` | Número de episodios de hospitalización en los 12 meses previos | int | HIS / sintético | Sí | Proxy de cronicidad; valores > 5 son outliers a revisar |
| `estancia_dias` | Días de hospitalización del episodio | int | EMH / sintético | Sí | Valores 0 posibles en altas del mismo día; no son errores |
| `ratio_estancia_grd` | Cociente entre estancia real y estancia media del GRD | float | Derivado | No (derivada) | Valores > 2 indican estancias prolongadas; < 0.5 posible alta precoz |
| `biomarcador_troponina` | Nivel de troponina en sangre (µg/L) | float | LIS / sintético | No | -1 si no se realizó la prueba; no confundir con nulo |
| `biomarcador_pct` | Nivel de procalcitonina (ng/mL), marcador de infección bacteriana | float | LIS / sintético | No | -1 si no aplica; valores > 0.5 ng/mL indican riesgo elevado |
| `biomarcador_bnp` | BNP (pg/mL), marcador de insuficiencia cardíaca | float | LIS / sintético | No | -1 si no aplica; valores > 100 pg/mL clínicamente significativos |
| `pendencias_diagnosticas` | Indica si hay pruebas diagnósticas sin resultado al momento del alta | int (0/1) | HIS / sintético | Sí | Variable de riesgo de alta prematura |
| `reingreso_30d` | Variable objetivo M3: reingreso hospitalario en los 30 días posteriores al alta | int (0/1) | EMH / sintético | Sí | Clase desbalanceada (~10-15% positivos) |
| `grd_codigo` | Código del Grupo Relacionado por el Diagnóstico | str | GRD SNS | Sí | Usar versión GRD-SNS fijada en el proyecto |
| `coste_medio_euros` | Coste medio por episodio según tarifa GRD del SNS | float | GRD SNS | Sí | Puede variar entre CCAA; usar tarifa nacional como referencia |
| `peso_relativo` | Peso relativo del GRD como proxy de complejidad clínica | float | GRD SNS | Sí | Útil como variable predictora en M3 |
| `latitud` / `longitud` | Coordenadas del centro hospitalario | float | ESCRI / geolocalización | Sí (M2) | Necesarias para calcular tiempo de traslado en M2 |
| `indicador_estacional` | Variable binaria: 1 si la semana cae en otoño-invierno (sem. 40-12) | int (0/1) | Derivado del calendario | No (derivada) | Captura la estacionalidad de patología respiratoria y cardiovascular |
| `ratio_urgentes_programados` | Cociente entre ingresos urgentes y programados en la semana | float | Derivado | No (derivada) | Indicador de presión; valores > 2 implican alto estrés operativo |

---

## 7. Problemas de calidad esperados

### 7.1 Valores nulos

- La EMH no desagrega por servicio hospitalario específico, solo por diagnóstico y CCAA. La asignación de ingresos a servicio requiere una regla de mapeo diagnóstico → servicio que puede generar nulos cuando el diagnóstico no cae en ninguna especialidad convencional (ej. códigos Z de factores que influyen en el estado de salud).
- Los biomarcadores (troponina, PCT, BNP) no están presentes en todos los episodios. La ausencia no es un error de datos sino una realidad clínica: no a todos los pacientes se les solicita la misma batería de pruebas. Se usará el valor centinela -1 en lugar de nulo para preservar esta distinción.

### 7.2 Duplicados

- La ESCRI puede presentar registros duplicados para hospitales que cambiaron de nombre o de dependencia funcional (traslado de titularidad pública a privada o viceversa) en el período 2014-2023. Se verificará mediante el código de centro oficial.
- El dataset sintético puede generar duplicados exactos si el proceso de simulación no incluye una semilla aleatoria fija. Se añadirá una semilla reproducible y se aplicará deduplicación por `episodio_id`.

### 7.3 Inconsistencias entre fuentes

- La EMH y la ESCRI usan distintos niveles de agregación geográfica. La EMH agrega por CCAA mientras que la ESCRI agrega por centro. El cruce requiere una tabla de correspondencia hospital → CCAA que debe construirse y mantenerse manualmente.
- Los códigos CIE-10 de la EMH pueden estar en versión española (CIE-10-ES) y el catálogo GRD en versión internacional. Se revisará la correspondencia en los diagnósticos más frecuentes del dataset.

### 7.4 Fechas y períodos

- La EMH publica los datos con un desfase de entre 12 y 18 meses respecto al año de referencia. El año 2023 puede estar incompleto o publicado en edición provisional a la fecha de inicio del proyecto.
- La semana epidemiológica no coincide exactamente con la semana calendario en el cambio de año (semana 1 puede comenzar en diciembre del año anterior). Esto debe tratarse con cuidado en los modelos de forecasting para no introducir discontinuidades artificiales.

### 7.5 Outliers

- Valores de ocupación superiores al 100% son técnicamente posibles (camas supletorias) y no deben eliminarse automáticamente; son exactamente los casos de interés analítico para el modelo.
- La estancia media puede presentar valores extremos en patologías crónicas complejas (ej. rehabilitación neurológica con estancias de varios meses). Se analizarán los percentiles 95 y 99 por GRD antes de decidir si se limitan mediante winsorizing o se mantienen como información legítima.
- El índice de Charlson en el dataset sintético puede presentar valores implausibles si la distribución de comorbilidades no está correctamente calibrada. Se validará la distribución empírica contra la literatura de referencia.

### 7.6 Desequilibrio de clases

- La variable objetivo `reingreso_30d` presenta un desequilibrio severo: aproximadamente el 10-15% de los episodios resultan en reingreso a 30 días. Esto no es un problema de calidad de datos sino una característica inherente del fenómeno que debe gestionarse en el pipeline de modelado (SMOTE, ajuste de pesos de clase, umbral de decisión adaptado).

### 7.7 Cambios de definición entre versiones anuales

- La ESCRI ha modificado la definición de algunas variables (especialmente dotación tecnológica) en el período 2014-2023. Se revisará la nota metodológica anual del Ministerio para identificar rupturas de serie.

---

## 8. Decisiones de limpieza y transformación previstas

### Tratamiento de nulos

- **Biomarcadores ausentes:** valor centinela -1 (no nulo) para distinguir "no solicitado" de "dato faltante". Se creará una variable binaria auxiliar `_disponible` para cada biomarcador.
- **Variables de ocupación y estancia con nulos puntuales:** imputación por la mediana del hospital en el mismo año. Si el hospital no tiene ningún valor para una variable en todo el año, se excluirá ese registro.
- **Diagnóstico CIE-10 nulo:** exclusión del registro, ya que es variable obligatoria para la construcción del GRD y del módulo de scoring.

### Eliminación de duplicados

- Deduplicación en `gold_episodios_alta` por `episodio_id`. En caso de duplicados exactos, se conserva el primer registro.
- En `gold_red_hospitalaria`, deduplicación por `(hospital_id, anio)`. Si un hospital aparece con dos nombres distintos en el mismo año, se conserva el registro con el código ESCRI oficial y se documenta la incidencia.

### Normalización de formatos

- Fechas: formato ISO 8601 (`YYYY-MM-DD`). La semana epidemiológica se expresa como entero (1-53).
- Nombres de columnas: convención `snake_case` en minúsculas, sin tildes ni caracteres especiales.
- Códigos CIE-10: normalización a formato estándar `X##.#` sin puntos intermedios adicionales. Se eliminarán los espacios y se forzará mayúsculas.
- Categorías textuales (tipo de ingreso, tipo de centro): se definirá un vocabulario controlado y se mapearán las variantes encontradas en los datos crudos (ej. "Urgente", "urgente", "URGENTE" → "urgente").

### Variables derivadas a construir

| Variable derivada | Cálculo | Dataset destino |
|---|---|---|
| `ratio_estancia_grd` | `estancia_dias / estancia_media_grd` | `gold_episodios_alta` |
| `ratio_urgentes_programados` | `ingresos_urgentes / ingresos_programados` | `gold_demanda_asistencial` |
| `tendencia_ocupacion_7d` | Diferencia de `ocupacion_pct` respecto a las 7 semanas anteriores | `gold_demanda_asistencial` |
| `indicador_estacional` | 1 si `semana_epidemiologica` ∈ [40, 53] ∪ [1, 12], 0 en otro caso | `gold_demanda_asistencial` |
| `_disponible` (por biomarcador) | 1 si el valor del biomarcador ≠ -1, 0 en otro caso | `gold_episodios_alta` |
| `peso_relativo` | Join con `gold_referencia_grd` sobre `grp_grd` | `gold_episodios_alta` |

### Datos que se descartarán

- Registros de hospitales con menos de 50 camas en funcionamiento: son centros de larga estancia o sociosanitarios que no responden al problema asistencial modelado.
- Episodios con estancia de 0 días y diagnóstico de cirugía mayor ambulatoria (CMA): son altas del mismo día que no comparten el patrón de riesgo de reingreso del modelo.
- Años 2020 y 2021 de la serie de demanda asistencial: la pandemia de COVID-19 genera una ruptura estructural en los patrones de ingreso que contaminaría el entrenamiento del modelo de forecasting. Se documentará la exclusión y se evaluará en el EDA si su inclusión como caso especial aporta valor.

### Criterio de validez de un registro

Un registro se considerará válido si cumple todas las condiciones siguientes:
- Tiene diagnóstico principal CIE-10 válido y mapeado a GRD.
- La estancia registrada es mayor o igual a 1 día (para episodios de scoring de alta).
- Los campos obligatorios del diccionario tienen valor no nulo.
- No es un duplicado según la clave primaria del dataset.

---

## 9. Riesgos del modelo de datos

### ¿Qué parte del modelo de datos está más clara?

La estructura de la capa gold para el Módulo 1 (demanda asistencial) es la más sólida. La EMH y la ESCRI son fuentes públicas bien documentadas, con series históricas largas y metodología estable, y el problema de forecasting con granularidad semanal encaja perfectamente con el volumen y estructura de esos datos.

### ¿Qué parte genera más incertidumbre?

El dataset sintético del Módulo 3 es el elemento más incierto del modelo. Su calidad depende de que las distribuciones estadísticas usadas para generarlo representen fielmente la realidad clínica. Si las distribuciones de comorbilidades, biomarcadores o patrones de reingreso no están bien calibradas, los modelos entrenados sobre él no serán generalizables, lo que limitaría el valor clínico de las predicciones aunque las métricas de validación cruzada sean buenas.

### ¿Qué fuente o tabla puede dar más problemas?

La tabla `gold_red_hospitalaria` depende del cruce entre la ESCRI (datos de ocupación y dotación) y una fuente de geolocalización de centros. Este cruce requiere una tabla de correspondencia hospital → coordenadas que no está disponible directamente en ninguna fuente oficial y deberá construirse manualmente o a partir del Registro de Centros del Ministerio de Sanidad. Es el punto de mayor riesgo operativo.

### ¿Qué ocurriría si no podéis construir la capa gold tal como la habéis definido?

- Si el dataset sintético no alcanza la calidad mínima para el Módulo 3, se puede sustituir por un análisis puramente descriptivo del riesgo de reingreso sobre los datos agregados de la EMH, sin modelo predictivo individual.
- Si la geolocalización de centros no es viable, el Módulo 2 puede simplificarse a un algoritmo de derivación basado únicamente en ocupación y capacidad por CCAA, sin componente geográfico.
- Si la serie ESCRI presenta demasiadas rupturas metodológicas, se puede restringir la ventana temporal a 2018-2023, período con mayor homogeneidad en la definición de indicadores.

### ¿Qué alternativa tendríais para simplificar el modelo si fuera necesario?

La simplificación más razonable sería reducir el proyecto a un único dataset gold (`gold_episodios_alta`) y un único módulo (Módulo 3 de scoring de alta), que es el núcleo predictivo con mayor valor clínico y el que mejor se apoya en datos públicos directamente disponibles. El Módulo 1 y el Módulo 2 pueden quedar como propuesta arquitectónica sin implementación completa si el tiempo o los datos no lo permiten.
