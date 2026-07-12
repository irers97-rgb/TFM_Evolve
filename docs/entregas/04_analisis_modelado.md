# Entrega 4 - Diseño del análisis y estrategia de modelado

---

## 1. Problema que se busca resolver

### Qué ocurre actualmente

El Sistema Nacional de Salud español opera con frecuencia bajo presión asistencial elevada, lo que condiciona dos decisiones clínico-operativas críticas:

- **Altas prematuras:** cuando la presión de camas es alta, la decisión de alta se ve influida por la necesidad de liberar recursos más que por el estado clínico real del paciente. Esto se traduce en **reingresos evitables del 20-25%** a 30 días en patologías crónicas de alta prevalencia como la insuficiencia cardíaca (INE, 2023), con un coste estimado superior a 55 millones de euros anuales solo en esa categoría diagnóstica.
- **Derivación reactiva:** la redistribución de pacientes entre centros de la red hospitalaria se activa mayoritariamente cuando el centro ya está saturado, no de forma anticipada. En 2023 se registraron **283.792 traslados no programados** (5,83% del total de altas), con mayor riesgo clínico y coste operativo duplicado respecto a una derivación planificada.

Ambos problemas comparten la misma causa raíz: la ausencia de herramientas basadas en datos que aporten objetividad, anticipación y homogeneidad a decisiones que hoy dependen en gran medida del criterio individual y de la presión del momento.

### Quién utiliza el resultado y para qué decisión

| Usuario final | Decisión o acción que apoya el sistema |
|---|---|
| **Dirección médica / gestión de camas** | Anticipar picos de demanda a 24-72h para planificar dotación de camas y personal antes de que se produzca la saturación. |
| **Médicos especialistas en planta** | Consultar el score de riesgo de reingreso a 30 días antes de confirmar el alta de un paciente, como capa adicional de apoyo al criterio clínico. |
| **Gestores de camas / coordinación de red** | Consultar el ranking de centros receptores recomendados cuando se necesita derivar a un paciente, en función de disponibilidad, compatibilidad clínica y distancia. |

El sistema **no sustituye el criterio clínico**: añade una capa de evidencia cuantitativa, explicable y trazable a decisiones que hoy se toman de forma mayoritariamente reactiva.

### Qué debe producir el proyecto para considerarse útil

El MVP final es un **dashboard interactivo** con cuatro bloques funcionales:

1. **EDA:** exploración de la serie histórica 2014-2023 (demanda, reingresos, traslados, ocupación) segmentada por patología y CCAA.
2. **Forecasting de demanda:** proyección de ingresos urgentes a 24-72h por servicio/CCAA.
3. **Ranking de derivación:** puntuación y orden de centros candidatos ante un escenario de saturación.
4. **Scoring de riesgo de alta:** probabilidad de reingreso a 30 días por episodio, con explicación de los factores que la determinan.

Se considerará útil si cada bloque produce una salida **accionable en menos de un clic** (sin necesidad de interpretación estadística por parte del usuario clínico) y si el scoring y el forecasting **superan de forma consistente a sus respectivos baselines** (ver punto 7).

---

## 2. Análisis de datos planteado (EDA) y utilidad esperada

### Preguntas de negocio críticas

- ¿Existe estacionalidad marcada en los ingresos urgentes? ¿Coincide con patrones respiratorios/cardiovasculares de otoño-invierno?
- ¿Qué GRDs presentan mayor desviación entre estancia real y estancia media de referencia (`ratio_estancia_grd`)? ¿Se concentran en determinadas CCAA o servicios?
- ¿Qué variables clínicas y operativas están más correlacionadas con `reingreso_30d`?
- ¿Cómo se distribuye la ocupación hospitalaria entre centros de una misma CCAA? ¿Hay desequilibrios estructurales que el módulo de derivación deba corregir?
- ¿El desbalanceo de la clase objetivo (10-15% positivos) es homogéneo entre patologías, edades y CCAA, o hay subgrupos de riesgo especialmente desatendidos?

### Análisis específicos por bloque

**Bloque 1 — Análisis temporal (soporte al forecasting, `gold_demanda_asistencial`)**

- Descomposición de la serie (tendencia, estacionalidad, residuo) por servicio y CCAA.
- Test de estacionariedad (Dickey-Fuller aumentado) para decidir si es necesario diferenciar la serie antes de modelos tipo SARIMA.
- Autocorrelación (ACF/PACF) para identificar la estructura de dependencia temporal y orientar el orden `(p,d,q)` del baseline estadístico.
- Análisis de ruptura estructural en 2020-2021 para decidir tratamiento (exclusión vs. variable exógena).
- Relación entre `indicador_estacional`, `festivo_semana` y `ingresos_urgentes`.

**Bloque 2 — Correlación y desbalanceo (soporte al scoring de alta, `gold_episodios_alta`)**

- Matriz de correlación entre variables numéricas (`charlson_index`, `ratio_estancia_grd`, `num_ingresos_previos_12m`, biomarcadores) y la variable objetivo.
- Análisis de la proporción de clase positiva (`reingreso_30d`) global y por subgrupos (edad, sexo, CCAA, GRD), para dimensionar la estrategia de rebalanceo.
- Distribución de biomarcadores diferenciando valor centinela (-1, "no solicitado") de valor real, evitando que el modelo interprete -1 como un valor clínico bajo.
- Boxplots de `ratio_estancia_grd` por resultado (reingreso sí/no) para validar su capacidad discriminante como feature.

**Bloque 3 — Análisis geográfico y de capacidad (soporte a la derivación, `gold_red_hospitalaria`)**

- Distribución de ocupación media anual y camas UCI por CCAA y tipo de centro.
- Mapeo de distancias entre centros (a partir de `latitud`/`longitud`) para estimar tiempos de traslado orientativos.
- Relación entre `presion_urgencias_pct` y `indice_rotacion` para identificar centros estructuralmente saturados frente a picos puntuales.

### Cómo guía el EDA el feature engineering

El EDA no es un ejercicio descriptivo aislado: cada hallazgo se traduce directamente en una decisión de modelado. La estacionariedad determina si se diferencia la serie antes de SARIMA; la correlación y el desbalanceo determinan la estrategia de preprocesado del Módulo 3 (pesos de clase/SMOTE, selección de features); y la dispersión geográfica y de capacidad alimenta directamente los pesos del algoritmo multicriterio del Módulo 2. Los indicadores y gráficos generados en esta fase se reutilizan como componentes del bloque EDA del dashboard final.

---

## 3. Tipo de modelos que se van a plantear

El proyecto tiene una lógica analítica mixta: dos módulos con modelado predictivo (forecasting y clasificación) y un módulo basado en lógica de reglas/optimización multicriterio.

### Módulo 1 — Forecasting de demanda

| Alternativa | Tipo | Por qué se plantea | Limitación principal |
|---|---|---|---|
| **Baseline** | Naive estacional / media móvil (misma semana año anterior, o media de últimas 4 semanas) | Referencia mínima, coste computacional nulo, fácil de explicar | No captura tendencia ni eventos exógenos (festivos, olas epidémicas) |
| **Candidato 1** | Prophet / SARIMA | Modelos interpretables de series temporales, manejan estacionalidad múltiple y son robustos con series relativamente cortas (10 años semanales) | Prophet puede sobre-suavizar picos agudos; SARIMA requiere series estacionarias y ajuste manual de órdenes |
| **Candidato 2** | LSTM | Puede capturar patrones no lineales y dependencias complejas entre servicios/CCAA si se dispone de suficiente histórico | Mayor coste computacional, menor interpretabilidad, riesgo de sobreajuste con series de tamaño moderado |

### Módulo 2 — Derivación inteligente

Este módulo **no requiere un modelo predictivo tradicional**. Se plantea un **algoritmo de puntuación ponderada multicriterio** (y, como extensión opcional, una formulación de optimización lineal si el tiempo del curso lo permite) que combine, para cada centro candidato: disponibilidad de camas especializadas, compatibilidad tecnológica con el perfil diagnóstico, tiempo de traslado estimado y nivel de ocupación actual.

**Justificación de no usar un modelo predictivo:** no existe una variable objetivo histórica fiable de "mejor derivación" sobre la que entrenar (las derivaciones pasadas son en su mayoría reactivas y subóptimas por definición, por lo que aprender de ellas replicaría el sesgo que el proyecto busca corregir). Un sistema de reglas ponderadas y auditable es, además, más adecuado para un contexto clínico donde la transparencia del criterio de decisión es tan importante como su precisión.

### Módulo 3 — Scoring de alta (riesgo de reingreso a 30 días)

| Alternativa | Tipo | Por qué se plantea | Limitación principal |
|---|---|---|---|
| **Baseline** | Clase mayoritaria / Regresión logística simple | Referencia mínima; la regresión logística además aporta interpretabilidad directa de coeficientes | Clase mayoritaria no aporta valor real de negocio; la logística puede no capturar interacciones no lineales entre comorbilidad y biomarcadores |
| **Candidato avanzado** | XGBoost + SHAP (explicabilidad local) | Buen rendimiento en datos tabulares con desbalanceo, manejo nativo de valores centinela/missing, y SHAP permite justificar cada predicción al médico de planta | Mayor coste de tuning y de mantenimiento; riesgo de sobreajuste si no se controla la profundidad/regularización |

---

## 4. Datos de entrada del análisis y los modelos

Se utiliza exclusivamente la capa gold definida en la Entrega 3, sin introducir nuevas fuentes.

### Mapeo por módulo

| Entrada | Descripción | Granularidad | Uso en el análisis o modelo |
|---|---|---|---|
| `gold_demanda_asistencial.csv` | Serie semanal de actividad hospitalaria por servicio y CCAA | Una fila por semana × servicio × CCAA | Entrada única del Módulo 1 (forecasting) |
| `gold_red_hospitalaria.csv` | Catálogo de hospitales con capacidad y ocupación | Una fila por hospital × año | Entrada única del Módulo 2 (derivación) |
| `gold_episodios_alta.csv` | Episodios de alta con variables clínicas y objetivo `reingreso_30d` | Una fila por episodio de alta | Entrada principal del Módulo 3 (scoring) |
| `gold_referencia_grd.csv` | Coste y estancia media por GRD | Una fila por código GRD | Tabla auxiliar (join) para M2 y M3 |

### Variables de entrada principales y transformaciones

- **Módulo 1:** `ingresos_urgentes` (objetivo), `ocupacion_pct`, `indicador_estacional`, `festivo_semana`, `ratio_urgentes_programados`, `tendencia_ocupacion_7d`. Transformación: diferenciación si la serie no es estacionaria; codificación cíclica de `semana_epidemiologica` para el candidato LSTM.
- **Módulo 2:** `camas_totales`, `camas_uci`, `ocupacion_media_anual_pct`, `presion_urgencias_pct`, `dispone_uci`, `dispone_hemodinamica`, `latitud`/`longitud`. No requiere transformación estadística, sino normalización de escalas para la puntuación ponderada.
- **Módulo 3:** `charlson_index`, `ratio_estancia_grd` (variable calculada como `estancia_dias / estancia_media_grd`), `num_ingresos_previos_12m`, biomarcadores (`biomarcador_troponina`, `biomarcador_pct`, `biomarcador_bnp` con **valor centinela -1** para "no solicitado", junto a su flag binario `_disponible`), `pendencias_diagnosticas`, `constantes_estables_48h`, `diagnostico_principal_cie10` (codificación por capítulo CIE-10 para reducir cardinalidad), `peso_relativo` (vía join con `gold_referencia_grd`).

### Variables que se descartan y motivo

| Variable / periodo | Motivo de exclusión |
|---|---|
| Años **2020-2021** en `gold_demanda_asistencial` | Ruptura estructural por la pandemia; contaminaría el aprendizaje de estacionalidad y tendencia del Módulo 1 |
| `exitus` como feature en Módulo 3 | Es un desenlace del propio episodio, no una variable disponible de forma consistente antes del alta; riesgo de fuga de información |
| `traslado` como feature en Módulo 3 | Puede estar codificado en el mismo momento que el alta y confundirse con la variable objetivo; se trata por separado |
| `episodio_id` | Identificador sin valor predictivo, solo trazabilidad |
| Episodios con estancia 0 días y CMA (cirugía mayor ambulatoria) | Ya excluidos en la capa gold (Entrega 3); no comparten el patrón de riesgo del resto de episodios |

### Disponibilidad real en el momento de la predicción

Todas las variables seleccionadas para el Módulo 3 deben estar cerradas **antes** de la decisión de alta (comorbilidades, biomarcadores solicitados durante el ingreso, estancia acumulada hasta ese momento). Se excluye explícitamente cualquier variable que solo se registre en el momento del alta o después (ver riesgo de leakage en el punto 8).

---

## 5. Datos de salida y forma de consumo

### Estructura común de salida

| Campo | Descripción | Tipo | Ejemplo |
|---|---|---|---|
| `id_entidad` | Identificador de la unidad predicha (episodio, hospital o combinación servicio-semana) | string / integer | `episodio_id`, `hospital_id` |
| `prediccion_score` | Resultado principal: probabilidad de reingreso, volumen de ingresos previsto o puntuación de derivación | float / categoría | `0.73` (riesgo alto) |
| `fecha_ejecucion` | Momento de generación del resultado | datetime | `2026-07-12T08:00:00` |
| `explicacion` | Factores que justifican el resultado | texto / estructura | Top-5 valores SHAP (M3); intervalo de confianza (M1); desglose de puntuación por criterio (M2) |

### Particularidades por módulo

- **Módulo 1:** salida a nivel semana × servicio × CCAA, con predicción puntual e **intervalo de confianza**, consumida como serie temporal graficada en el dashboard.
- **Módulo 2:** salida a nivel hospital candidato, con **ranking ordenado** y desglose de la puntuación por criterio (ocupación, compatibilidad, distancia), consumida como tabla ordenada con opción de "ver detalle".
- **Módulo 3:** salida a nivel episodio, con **score de riesgo (alto/moderado/bajo)** y **valores SHAP** de las variables más determinantes, consumida como tarjeta de paciente individual dentro del dashboard.

### Cómo mitiga el dashboard la fricción clínica

El dashboard traduce cada salida numérica en una recomendación de lectura rápida: semáforo de riesgo en vez de probabilidad cruda, ranking en vez de matriz de puntuaciones, y gráfico de barras SHAP en vez de tabla de coeficientes. El objetivo es que el usuario clínico entienda el resultado en segundos, sin necesitar formación estadística previa, preservando siempre la trazabilidad hasta la explicación subyacente si decide profundizar.

---

## 6. Estrategia para diseñar y seleccionar el modelo

### Pipeline de preprocesamiento (previo al modelado)

1. **Tratamiento del desbalanceo (Módulo 3):** combinación de ajuste de pesos de clase (`class_weight`/`scale_pos_weight` en XGBoost) como opción principal, y SMOTE como alternativa a evaluar solo sobre el conjunto de entrenamiento (nunca sobre validación/test, para evitar contaminación).
2. **Codificación de variables categóricas:** `diagnostico_principal_cie10` agrupado por capítulo CIE-10 antes de one-hot/target encoding; variables binarias ya codificadas en la capa gold.
3. **Imputación de nulos:** siguiendo las reglas ya fijadas en la Entrega 3 (mediana por hospital/año para variables operativas; exclusión de registros sin diagnóstico válido). Los biomarcadores **no se imputan**, se preserva el valor centinela -1 más su flag `_disponible`.
4. **Escalado:** estandarización de variables numéricas continuas para el baseline de regresión logística; no es estrictamente necesario para XGBoost, pero se mantiene consistente para comparabilidad de importancias.

### Construcción y comparación de alternativas

- Baseline simple por módulo (ver punto 3) entrenado primero, como referencia mínima obligatoria.
- Modelos candidatos entrenados sobre el mismo split para garantizar comparabilidad.
- Comparación en base a: calidad predictiva (métricas del punto 7), estabilidad temporal (variación de la métrica entre distintos folds/periodos), interpretabilidad (disponibilidad de SHAP o coeficientes), coste computacional (tiempo de entrenamiento/inferencia) y utilidad directa para el MVP (facilidad de mostrarlo en el dashboard).

### Regla de decisión final

Un modelo candidato **solo se selecciona** si cumple simultáneamente:

1. Supera al baseline en la métrica principal del módulo con un margen no trivial (ver criterio de aceptación, punto 7).
2. Mantiene un rendimiento **estable** entre periodos/folds (sin caídas bruscas en ningún segmento relevante de edad, sexo o CCAA).
3. Permite generar una **explicación interpretable por un médico no técnico** (SHAP en Módulo 3, intervalos en Módulo 1).
4. Es **viable de reentrenar y mantener** dentro del alcance temporal del curso.

La mejor métrica aislada **no es criterio suficiente**: un modelo ligeramente inferior en AUC/F1 pero más estable y explicable se preferirá frente a uno marginalmente mejor pero opaco o inestable.

---

## 7. Estrategia de validación y evaluación

| Elemento | Decisión prevista | Justificación |
|---|---|---|
| **Separación de datos — Módulo 1** | Split temporal estricto: backtesting con `TimeSeriesSplit` (entrenamiento en ventana expansiva, validación en las semanas siguientes) | Evita usar información futura para predecir el pasado; reproduce el uso real del forecasting |
| **Separación de datos — Módulo 3** | `Stratified K-Fold` sobre `reingreso_30d`, con partición adicional por `episodio_id` para evitar que un mismo paciente simulado aparezca en train y test si el generador sintético produce episodios repetidos | Mantiene la proporción de clase positiva en cada fold pese al desbalanceo; evita fuga entre partición de entrenamiento y prueba |
| **Métrica principal — Módulo 1** | MAE y RMSE por horizonte (24h/72h) | Miden directamente el error de predicción de volumen, interpretables en "número de ingresos de error" |
| **Métrica principal — Módulo 3** | Recall, F1-Score y curva Precision-Recall (PR-AUC) | Con una clase positiva del 10-15%, el accuracy es engañoso; el coste de un falso negativo (alta que termina en reingreso) es clínicamente mayor que el de un falso positivo, por lo que se prioriza recall sin descuidar precisión vía F1/PR-AUC |
| **Baseline de referencia** | Naive/media móvil (M1); clase mayoritaria y regresión logística (M3) | Permite cuantificar la mejora real aportada por el modelo candidato |
| **Criterio de aceptación** | M1: reducción de al menos 15-20% en MAE respecto al baseline naive. M3: mejora de al menos 15 puntos porcentuales en PR-AUC respecto al baseline de clase mayoritaria, manteniendo recall ≥ 0,70 en la clase positiva | Umbral mínimo para considerar que el modelo aporta valor de negocio real, no solo mejora estadística marginal |

### Análisis de errores y segmentos

Se revisará el rendimiento desagregado por CCAA, franja de edad y sexo (Módulo 3) para detectar sesgos sistemáticos, y por servicio/CCAA (Módulo 1) para detectar zonas con peor ajuste del forecasting. Los casos extremos (outliers de estancia, picos de ocupación >100%) se analizarán de forma cualitativa para verificar que no degradan el modelo de forma desproporcionada.

### Si ningún modelo alcanza el criterio de aceptación

Se documentará el resultado del baseline como entrega válida del MVP (más simple pero honesta), y se aplicará el plan de contingencia descrito en el punto 8.

---

## 8. Riesgos y alternativas

| Riesgo | Descripción | Plan de contingencia |
|---|---|---|
| **Fidelidad clínica del dataset sintético (Módulo 3)** | Si las distribuciones de comorbilidad, biomarcadores o reingreso no están bien calibradas frente a la literatura clínica, el modelo puede no ser generalizable aunque las métricas de validación cruzada sean buenas | Validar la distribución empírica del dataset sintético contra la literatura de referencia antes de entrenar; si la calidad no es suficiente, simplificar a un análisis descriptivo del riesgo sobre datos agregados de la EMH, sin modelo predictivo individual |
| **Disponibilidad de geolocalización (Módulo 2)** | La tabla hospital → coordenadas no está disponible directamente en ninguna fuente oficial y requiere construcción manual a partir del Registro de Centros | Si no es viable, simplificar el Módulo 2 a un algoritmo basado únicamente en ocupación y capacidad por CCAA, sin componente geográfico |
| **Data leakage en el Módulo 3** | Riesgo de incluir variables del episodio que en realidad se cierran o cambian justo en el momento del alta (p. ej. `traslado`, `exitus`, o biomarcadores solicitados después de la decisión clínica) | Excluir explícitamente del feature set cualquier variable no disponible con certeza antes de la decisión de alta (ver punto 4); auditar el pipeline de features con una revisión temporal explícita antes de entrenar |
| **Desbalanceo severo de la clase objetivo** | Solo 10-15% de positivos en `reingreso_30d`; riesgo de que el modelo optimice trivialmente por la clase mayoritaria | Uso de pesos de clase/SMOTE y métricas robustas al desbalanceo (recall, F1, PR-AUC) en vez de accuracy, como ya se define en el punto 7 |
| **Ruptura temporal 2020-2021 (Módulo 1)** | La pandemia introduce un cambio estructural en la demanda que puede distorsionar el aprendizaje de estacionalidad | Exclusión del periodo del entrenamiento (ya decidido en la Entrega 3), evaluando en el EDA si aporta valor como variable exógena |
| **Heterogeneidad entre CCAA en codificación/reporte** | Puede introducir ruido o sesgos de cobertura en el análisis geográfico y en el forecasting por CCAA | Normalización y armonización explícita en la capa processed; análisis de sensibilidad restringiendo a CCAA con series más completas si es necesario |

### Parte de la estrategia con mayor incertidumbre

La calibración del dataset sintético del Módulo 3 sigue siendo, como ya se identificó en la Entrega 2, el elemento de mayor riesgo del proyecto: es la pieza que sostiene el módulo de mayor valor clínico (scoring de alta) y la que menos se apoya en datos reales verificables a nivel individual.

### Alternativa global si los modelos no superan el baseline

Si ninguno de los modelos candidatos supera de forma robusta a su baseline, el proyecto se replegará a un **sistema de reglas clínicas validadas por literatura** (p. ej. umbrales de Charlson, ratio de estancia y biomarcadores conocidos en la bibliografía de predicción de reingreso), presentado igualmente en el dashboard como scoring, pero sustituyendo el modelo de machine learning por una lógica determinista y completamente auditable — consistente con la alternativa de simplificación ya prevista en la Entrega 3.
