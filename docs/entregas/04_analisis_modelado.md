# Entrega 4 - Diseño del análisis y estrategia de modelado

> **Nota de revisión (respuesta al feedback recibido):** el feedback señala dos desajustes concretos entre lo que se quería producir y los datos disponibles -una tabla semanal no puede justificar predicciones a 24-72 horas, y una ocupación media anual no representa la disponibilidad real de camas para recomendar derivaciones-, un problema metodológico grave en el planteamiento de scoring por episodio (separar por `episodio_id` no evita que varios episodios del mismo paciente aparezcan a la vez en entrenamiento y prueba; hace falta un identificador de paciente y validación por grupos, preferiblemente también temporal), y el riesgo de entrenar y validar con reingresos sintéticos, que puede llevar al modelo a reproducir las reglas usadas para generar la etiqueta en vez de demostrar validez clínica real. También pide priorizar un único módulo y alinear de forma estricta su granularidad, horizonte y fuente de verdad. Este documento aplica esos cuatro puntos: el análisis y modelado detallado se centra en el **Módulo 1 (predicción de demanda)**, con una verificación explícita de que su horizonte (4-8 semanas) es coherente con su granularidad (semanal) - nunca se plantean predicciones a 24-72 horas sobre un dato semanal. Los Módulos 2 y 3 pasan a un anexo de diseño conceptual, no implementado, en el que se incorporan explícitamente las correcciones metodológicas señaladas (identificador de paciente y validación por grupos/temporal, y el riesgo de circularidad con etiquetas sintéticas) por si en el futuro se activan.

---

## 1. Problema que se busca resolver

### Qué ocurre actualmente

Los hospitales del SNS gestionan de forma mayoritariamente reactiva la planificación de la demanda asistencial. La ausencia de herramientas basadas en datos reales dificulta anticipar picos de ingresos con margen suficiente para planificar dotación de personal y camas.

### Quién utiliza el resultado y para qué decisión

| Módulo | Usuario final | Decisión que apoya | Estado |
|---|---|---|---|
| **M1 - Demanda** | Dirección médica / gestión de camas | Planificar dotación con 4-8 semanas de antelación | **MVP de este entregable** |
| M2 - Derivación | Dirección médica de área sanitaria / planificación de red | Priorizar en qué patologías/CCAA reforzar capacidad especializada, de cara a la temporada | Diseño conceptual, ver Anexo |
| M3 - Riesgo de reingreso | Dirección médica / calidad asistencial | Priorizar en qué segmentos reforzar protocolos de alta o seguimiento post-alta | Diseño conceptual, ver Anexo |

El Módulo 1 no opera en tiempo real: la fuente (EMH) se publica con retraso y a granularidad agregada. Su valor está en la **planificación estructural a semanas vista**, no en la alerta operativa del día a día - por eso el horizonte se mantiene en 4-8 semanas y en ningún momento se plantea un horizonte de 24-72 horas, que sí exigiría un dato con actualización diaria u horaria que la EMH no ofrece.

### Qué debe producir el proyecto para considerarse útil

Un dashboard con dos bloques: EDA histórico y forecasting de demanda (M1). Se considera útil si M1 supera al baseline naive. Los paneles de M2/M3 no forman parte del criterio de éxito de este entregable.

---

## 2. Análisis de datos planteado (EDA) y utilidad esperada

### Preguntas de negocio críticas (M1)

- Estacionalidad de ingresos urgentes.
- Diferencias entre CCAA (y entre servicios, si la Entrega 3 confirma esa desagregación).
- Estacionariedad de la serie.
- Tratamiento de la ruptura 2020-2021.

### Análisis específicos (M1)

- Descomposición estacional.
- Test de Dickey-Fuller aumentado (ADF) para estacionariedad.
- ACF/PACF para identificar orden del modelo.

### Cómo guía el EDA el feature engineering

El EDA determina si la serie necesita diferenciación, qué variables exógenas (festivos, indicador estacional) aportan señal real, y confirma si la desagregación por servicio (pendiente de validar en la Entrega 3) es lo bastante estable como para modelarse por separado o si conviene agregarla a nivel CCAA.

Las preguntas de EDA para Módulos 2 y 3 se mantienen en el Anexo, sin ejecutarse en este entregable.

---

## 3. Tipo de modelo que se va a plantear

| Módulo | Alternativa | Tipo | Por qué se plantea | Limitación principal |
|---|---|---|---|---|
| **M1** | Baseline naive + SARIMA/Prophet | Forecasting de series temporales | Único módulo del MVP; horizonte de 4-8 semanas coherente con la granularidad semanal de la fuente | Prophet puede sobre-suavizar picos; SARIMA exige estacionariedad |

El planteamiento de modelo para M2 y M3 (regresión sobre tasas agregadas, con explicabilidad SHAP en M3) se conserva en el Anexo como diseño conceptual, con las correcciones metodológicas que exige la revisión, pero no se implementa en este entregable.

---

## 4. Datos de entrada del análisis y el modelo (M1)

`gold_demanda_asistencial.csv` (ver Entrega 3, sección 5.1).

### Variables que se descartan y motivo

| Variable / periodo | Motivo de exclusión |
|---|---|
| Años 2020-2021 | Ruptura estructural pandémica |
| Ocupación diaria por centro | No disponible en fuentes públicas (ESCRI es foto anual, no serie) - por eso el horizonte se queda en semanas, nunca en 24-72 horas |
| Desagregación por servicio | Se incluye solo si la Entrega 3 confirma que la fuente la soporta realmente; si no, el modelo opera a nivel CCAA |

---

## 5. Datos de salida y forma de consumo (M1)

Salida sin cambios respecto al diseño ya validado: proyección semanal de `ingresos_urgentes` por CCAA (y servicio si se confirma), con intervalo de confianza, consumida en el dashboard de la Entrega 5.

---

## 6. Estrategia de validación y evaluación (M1)

| Elemento | M1 |
|---|---|
| **Separación de datos** | Split temporal (`TimeSeriesSplit`) |
| **Métrica principal** | MAE, RMSE por horizonte |
| **Baseline de referencia** | Naive estacional |
| **Criterio de aceptación** | Reducción ≥15-20% de MAE vs. baseline (horizonte 4 semanas) |

### Verificación explícita de alineación granularidad-horizonte-fuente de verdad

Este es el punto que la revisión pide comprobar de forma estricta: la fuente (EMH) tiene granularidad semanal; el horizonte de predicción es de 4-8 semanas; y la fuente de verdad contra la que se evalúa (`ingresos_urgentes` real de esa semana) existe a esa misma granularidad. No hay ningún punto del diseño en que se prometa una predicción a 24-72 horas - eso exigiría una fuente con actualización diaria que no está disponible, y de plantearse en el futuro sería un módulo distinto, con su propia fuente y su propia validación, no una lectura más fina de este dataset semanal.

### Si el módulo no alcanza el criterio de aceptación

Se documenta como resultado válido del TFM el panel descriptivo (ranking histórico sin componente predictivo añadido).

---

## 7. Riesgos y alternativas (M1)

| Riesgo | Descripción | Plan de contingencia |
|---|---|---|
| **Acceso a microdatos EMH** | Sujeto a solicitud especial, no descarga directa (ver Entrega 3, sección 2) | Priorizar agregados públicos ya disponibles si sostienen la granularidad necesaria; documentar el trámite como riesgo de calendario si no |
| **Desagregación por servicio no confirmada** | La Entrega 3 puede concluir que la EMH no soporta ese nivel | Modelo a nivel CCAA únicamente, documentando la pérdida de resolución |
| **Heterogeneidad de codificación entre CCAA** | Ruido en la serie | Normalización adicional en preparación de datos |

### Parte de la estrategia con mayor incertidumbre
Si la EMH soporta realmente la desagregación por servicio; de ello depende si M1 se entrega a nivel semana × CCAA × servicio o se simplifica a semana × CCAA.

---

## Anexo — Diseño conceptual de Módulos 2 y 3 (no implementado en este entregable)

> Se conserva el trabajo de diseño ya hecho sobre M2 y M3, incorporando las correcciones metodológicas señaladas en la revisión, para que quede documentado si en el futuro se activan (ver condiciones en la Entrega 3, sección 10.5). **Nada de esta sección se implementa, entrena o valida en el MVP actual.**

### A.1 Preguntas de negocio (EDA no ejecutado)

**Módulo 2:**
- ¿Qué patologías concentran mayor tasa de traslado sobre sus altas totales? ¿Se mantiene estable en el tiempo o varía por año?
- ¿La tasa de traslado correlaciona más con la severidad clínica (mortalidad, desviación de estancia) o con la capacidad estructural de la CCAA (camas totales)? - importante para no confundir "complejidad clínica" con "falta de camas".
- ¿Hay CCAA con tasas de traslado sistemáticamente altas independientemente de la patología?

**Módulo 3:**
- ¿Qué combinaciones patología × edad presentan mayor `tasa_reingreso_30d` según iCMBD?
- ¿La desviación de estancia correlaciona con mayor tasa de reingreso?
- ¿Existen diferencias sistemáticas entre CCAA en la tasa de reingreso a igualdad de patología y edad?

### A.2 Tipo de modelo propuesto y correcciones metodológicas exigidas

| Módulo | Alternativa | Corrección incorporada tras la revisión |
|---|---|---|
| **M2** | Regresión (Ridge/Gradient Boosting Regressor) sobre `tasa_traslado` | La variable `camas_totales_ccaa` es contexto estructural anual; **nunca** debe leerse ni comunicarse como disponibilidad real de camas para recomendar una derivación concreta. El score es un índice de priorización de planificación, no una recomendación operativa |
| **M3** | Regresión (Gradient Boosting Regressor) sobre `tasa_reingreso_30d`, con SHAP | Dos correcciones obligatorias antes de poder implementarse: (1) si el diseño llegara a nivel de episodio individual, la separación entre entrenamiento y prueba debe hacerse por **identificador de paciente real**, no por `episodio_id` - separar por episodio no evita que varios episodios de la misma persona aparezcan a la vez en ambos conjuntos, lo que infla artificialmente las métricas; la validación debe ser por grupos (`GroupKFold` o equivalente sobre paciente) y preferiblemente también temporal (dejar fuera el último año). (2) Si la tasa de reingreso se completase alguna vez con dato sintético para ganar granularidad, existe el riesgo de que el modelo se limite a reproducir las reglas usadas para generar esa etiqueta, obteniendo métricas altas sin ninguna validez clínica real - por eso, mientras la variable objetivo no proceda de un indicador real agregado (iCMBD), M3 no se entrena |

Se descarta, como ya se documentó, un clasificador binario individual porque exigiría una etiqueta de reingreso por paciente que ninguna fuente pública ofrece sin recurrir a datos sintéticos.

### A.3 Datos de entrada (conceptual)

**M2:** `gold_presion_derivacion.csv`. Variable objetivo: `tasa_traslado`. Predictoras: `mortalidad_intrahosp_pct`, `desviacion_estancia`, `camas_totales_ccaa` (normalizada por altas totales de la CCAA), `anio`.

**M3:** `gold_riesgo_reingreso.csv`. Variable objetivo: `tasa_reingreso_30d`. Predictoras: `desviacion_estancia`, `mortalidad_intrahosp_pct`, `pct_ingreso_urgente`, `grupo_edad` (codificada ordinalmente), `patologia_cie10` (codificada, posiblemente agrupada).

### A.4 Datos de salida (conceptual)

**M2:**

| Campo | Descripción | Tipo |
|---|---|---|
| `patologia_cie10`, `ccaa_codigo`, `anio` | Unidad predicha | str/int |
| `score_presion_derivacion` | Índice de 0 a 100, mayor valor = mayor presión estructural de derivación (planificación, no recomendación individual) | float |
| `factores_principales` | 2-3 variables con mayor peso en el score | list[str] |

**M3:**

| Campo | Descripción | Tipo |
|---|---|---|
| `patologia_cie10`, `grupo_edad`, `ccaa_codigo`, `anio` | Unidad predicha | str/int |
| `score_riesgo_reingreso` | Índice de 0 a 100, mayor valor = mayor riesgo relativo de reingreso en ese segmento | float |
| `factores_principales` | 2-3 variables con mayor peso en el score | list[str] |

**Consumo previsto:** ranking/mapa por CCAA y patología, nunca una recomendación sobre un paciente individual identificado. El dashboard etiquetaría explícitamente estos paneles como "índices de priorización a nivel de segmento, no una evaluación de un paciente concreto".

### A.5 Estrategia de validación (conceptual, corregida)

| Elemento | M2 | M3 |
|---|---|---|
| **Separación de datos** | Leave-one-year-out (validación cruzada dejando un año fuera cada vez, dado el bajo número de años disponibles) | Leave-one-year-out; **si el diseño llegase a nivel de episodio, además validación por grupos sobre identificador de paciente real, nunca por `episodio_id`, preferiblemente combinada con la separación temporal** |
| **Métrica principal** | R² y MAE sobre `tasa_traslado` | R² y MAE sobre `tasa_reingreso_30d` |
| **Baseline de referencia** | Media histórica de `tasa_traslado` por patología (sin CCAA) | Media histórica de `tasa_reingreso_30d` por patología (sin CCAA ni edad) |
| **Criterio de aceptación** | Mejora de R² frente al baseline | Mismo criterio que M2 |

### A.6 Riesgos específicos de M2/M3 (conceptual)

| Riesgo | Descripción | Plan de contingencia |
|---|---|---|
| **Circularidad con etiqueta sintética (M3)** | Si la tasa de reingreso se completara con dato sintético, el modelo podría reproducir las reglas de generación de esa etiqueta en vez de demostrar validez clínica | M3 no se entrena mientras la variable objetivo no proceda de un indicador real agregado (iCMBD); si en algún momento se explorase con dato sintético por motivos exclusivamente técnicos, el resultado se presentaría como **validación técnica del prototipo**, nunca como evidencia de capacidad asistencial real |
| **Confusión entre severidad clínica y capacidad de red (M2)** | El modelo podría atribuir a "complejidad de la patología" lo que en realidad es falta de camas en una CCAA concreta | Análisis de varianza explícito en el EDA para separar ambos efectos antes de interpretar el score |
| **Fuga de información entre episodios del mismo paciente (M3, si llega a episodio)** | Separar por `episodio_id` no impide que dos episodios de la misma persona caigan uno en entrenamiento y otro en prueba, inflando la métrica | Validación por grupos sobre identificador de paciente real, combinada con separación temporal |
| **Granularidad real de iCMBD** | El indicador de reingreso puede no desagregar por patología y CCAA simultáneamente | Simplificar a la desagregación que sí ofrezca el indicador, documentando la pérdida de resolución |
| **Bajo volumen en combinaciones patología × CCAA (× edad)** | Tasas inestables, riesgo de sobreajuste | Umbral mínimo de altas por celda; agrupación de patologías poco frecuentes |

### A.7 Condición para activar este diseño

Ver Entrega 3, sección 10.5: solo se implementaría tras confirmar la granularidad real de GRD/iCMBD y con el MVP (M1) ya entregado, aplicando desde el primer momento las correcciones de validación (identificador de paciente real, validación por grupos y temporal) recogidas en A.2 y A.5.
