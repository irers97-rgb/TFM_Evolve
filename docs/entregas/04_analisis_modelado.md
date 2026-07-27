# Entrega 4 - Diseño del análisis y estrategia de modelado

> **Nota de revisión (post feedback sobre Entrega 4):** este documento reduce el alcance a un único módulo — predicción de demanda asistencial — y ajusta el horizonte, la granularidad y las métricas de éxito a lo que las fuentes públicas (EMH, Estadística de Centros de Atención Especializada) sostienen realmente, según la validación documentada en la Entrega 3 (sección 2). El módulo de scoring de alta y el de derivación quedan fuera del MVP: dependían de datos sintéticos (episodios, biomarcadores, reingreso) u ocupación diaria no disponible, lo que —tal como se señaló en el feedback— permitiría demostrar un pipeline pero no sostener conclusiones clínicas reales.

---

## 1. Problema que se busca resolver

### Qué ocurre actualmente

Los hospitales del Sistema Nacional de Salud gestionan la dotación de camas y personal de forma mayoritariamente reactiva ante picos de demanda estacional (otoño-invierno, olas epidémicas). La ausencia de una proyección de demanda con antelación suficiente dificulta la planificación de recursos y contribuye indirectamente a la presión asistencial que, en última instancia, condiciona decisiones clínicas como el alta o la derivación de pacientes.

### Quién utiliza el resultado y para qué decisión

| Usuario final | Decisión o acción que apoya el sistema |
|---|---|
| **Dirección médica / gestión de camas** | Planificar dotación de camas y personal con 4-8 semanas de antelación, apoyándose en el patrón histórico de demanda por servicio y CCAA. |

El sistema **no es una alerta operativa en tiempo real**: la fuente de datos (EMH) se publica con 12-18 meses de retraso, por lo que el valor del producto está en la **planificación estructural de temporada** (ej. anticipar el pico de invierno con semanas de antelación usando el patrón histórico), no en la gestión del día a día.

### Qué debe producir el proyecto para considerarse útil

El MVP final es un **dashboard interactivo** con dos bloques funcionales:

1. **EDA:** exploración de la serie histórica 2014-2023 de ingresos, segmentada por servicio y CCAA, con foco en estacionalidad y tendencia.
2. **Forecasting de demanda:** proyección semanal de ingresos urgentes y programados a 4-8 semanas vista, por servicio/CCAA, con intervalo de confianza.

Se considerará útil si la proyección **supera de forma consistente a un baseline naive** (ver punto 6) y si el resultado se presenta de forma que un gestor no técnico pueda interpretarlo sin necesitar formación estadística.

---

## 2. Análisis de datos planteado (EDA) y utilidad esperada

### Preguntas de negocio críticas

- ¿Existe estacionalidad marcada en los ingresos urgentes? ¿Coincide con patrones respiratorios/cardiovasculares de otoño-invierno?
- ¿Cómo varía la estacionalidad entre CCAA y servicios? ¿Hay patrones estructurales distintos que justifiquen modelos separados por región?
- ¿Qué relación hay entre `festivo_semana`, `indicador_estacional` y el volumen de ingresos urgentes?
- ¿Es la serie estacionaria? ¿Requiere diferenciación antes de un modelo tipo SARIMA?
- ¿Qué impacto tuvo la ruptura de 2020-2021 y cómo debe tratarse (exclusión vs. variable exógena)?

### Análisis específicos

- Descomposición de la serie (tendencia, estacionalidad, residuo) por servicio y CCAA.
- Test de estacionariedad (Dickey-Fuller aumentado) para decidir si diferenciar la serie antes del baseline estadístico.
- Autocorrelación (ACF/PACF) para orientar el orden `(p,d,q)` del modelo SARIMA.
- Análisis de ruptura estructural en 2020-2021.
- Relación entre `indicador_estacional`, `festivo_semana` e `ingresos_urgentes`.

### Cómo guía el EDA el feature engineering

La estacionariedad determina si se diferencia la serie antes de modelar; la fuerza y regularidad de la estacionalidad determina si el modelo trabaja a nivel nacional o desagregado por CCAA/servicio, en función de cuánto ruido introduce la desagregación. Los gráficos generados en esta fase se reutilizan como componentes del bloque EDA del dashboard final.

---

## 3. Tipo de modelo que se va a plantear

| Alternativa | Tipo | Por qué se plantea | Limitación principal |
|---|---|---|---|
| **Baseline** | Naive estacional / media móvil (misma semana año anterior, o media de últimas 4 semanas) | Referencia mínima, coste computacional nulo, fácil de explicar a un gestor no técnico | No captura tendencia ni eventos exógenos (festivos, olas epidémicas) |
| **Candidato** | SARIMA / Prophet | Modelos interpretables de series temporales, manejan estacionalidad múltiple y son robustos con series relativamente cortas (10 años semanales) | Prophet puede sobre-suavizar picos agudos; SARIMA requiere series estacionarias y ajuste de órdenes |

Se descarta un modelo tipo LSTM para el MVP: con una única fuente de granularidad semanal y sin datos exógenos en tiempo real, el coste de tuning y la menor interpretabilidad no se justifican frente a SARIMA/Prophet, que además son suficientes para el horizonte de 4-8 semanas planteado.

---

## 4. Datos de entrada del análisis y el modelo

Se utiliza exclusivamente `gold_demanda_asistencial.csv`, definido en la Entrega 3.

| Variable | Uso |
|---|---|
| `ingresos_urgentes` | Variable objetivo |
| `ingresos_programados` | Variable de contexto / feature |
| `ratio_urgentes_programados` | Feature derivada |
| `indicador_estacional`, `festivo_semana` | Features de estacionalidad |
| `camas_totales_ccaa` | Feature de contexto estructural (estática dentro del año) |
| `tendencia_ingresos_4s` | Feature de tendencia reciente |

**Transformación:** diferenciación si el test de Dickey-Fuller indica no estacionariedad; codificación cíclica de `semana_epidemiologica` si se explora algún modelo no lineal en el futuro.

### Variables que se descartan y motivo

| Variable / periodo | Motivo de exclusión |
|---|---|
| Años 2020-2021 | Ruptura estructural por la pandemia; contaminaría el aprendizaje de estacionalidad y tendencia |
| `ocupacion_pct` semanal | No existe con esa granularidad en las fuentes públicas (ver Entrega 3, sección 2); se sustituye por `camas_totales_ccaa` como contexto anual estático |

---

## 5. Datos de salida y forma de consumo

| Campo | Descripción | Tipo | Ejemplo |
|---|---|---|---|
| `semana_prediccion` | Semana objetivo de la proyección | int | Semana 42 de 2026 |
| `ccaa_codigo`, `servicio` | Unidad predicha | str | — |
| `prediccion_ingresos` | Volumen previsto de ingresos urgentes | float | 312 |
| `intervalo_confianza` | Rango de incertidumbre de la predicción | (float, float) | (280, 344) |
| `fecha_ejecucion` | Momento de generación del resultado | datetime | `2026-07-27T08:00:00` |

**Consumo:** serie temporal graficada en el dashboard, con la proyección superpuesta al histórico reciente y el intervalo de confianza visible, no solo el valor puntual.

### Cómo se evita presentar una estimación como una certeza

El dashboard muestra siempre el intervalo de confianza junto a la predicción puntual, y compara la predicción contra el error histórico del baseline, para que el usuario entienda el margen de error típico del modelo antes de planificar sobre él.

---

## 6. Estrategia de validación y evaluación

| Elemento | Decisión prevista | Justificación |
|---|---|---|
| **Separación de datos** | Split temporal estricto: backtesting con `TimeSeriesSplit` (entrenamiento en ventana expansiva, validación en las semanas siguientes) | Evita usar información futura para predecir el pasado; reproduce el uso real del forecasting |
| **Métrica principal** | MAE y RMSE por horizonte (2/4/8 semanas) | Miden directamente el error de predicción de volumen, interpretables en "número de ingresos de error" |
| **Baseline de referencia** | Naive estacional / media móvil | Permite cuantificar la mejora real aportada por el modelo candidato |
| **Criterio de aceptación** | Reducción de al menos 15-20% en MAE respecto al baseline naive, en al menos el horizonte de 4 semanas | Umbral mínimo para considerar que el modelo aporta valor real de planificación, no solo mejora estadística marginal |

### Análisis de errores y segmentos
Se revisará el rendimiento desagregado por CCAA y servicio para detectar zonas con peor ajuste, y se analizarán cualitativamente los picos extremos de demanda para verificar que no degradan el modelo de forma desproporcionada.

### Si el modelo no alcanza el criterio de aceptación
Se documentará el resultado del baseline como entrega válida del MVP (más simple pero honesta), reduciendo la promesa del producto a un panel descriptivo de estacionalidad sin componente predictivo, en línea con el plan de contingencia de la sección 7.

---

## 7. Riesgos y alternativas

| Riesgo | Descripción | Plan de contingencia |
|---|---|---|
| **Mapeo diagnóstico → servicio** | La EMH no desagrega nativamente por servicio; la regla de mapeo puede introducir ruido | Si el ruido es alto, se simplifica el modelo a nivel de CCAA sin desagregar por servicio |
| **Retraso de publicación de la EMH** | El último año de la serie puede estar en edición provisional; el dato nunca es "tiempo real" | Se documenta explícitamente como limitación de producto: el sistema es de planificación estructural, no de alerta operativa |
| **Ruptura temporal 2020-2021** | La pandemia introduce un cambio estructural en la demanda que puede distorsionar el aprendizaje de estacionalidad | Exclusión del periodo del entrenamiento, evaluando en el EDA si aporta valor como variable exógena |
| **Heterogeneidad entre CCAA** | Puede introducir ruido en el forecasting desagregado por CCAA | Análisis de sensibilidad restringiendo a CCAA con series más completas si es necesario |
| **Granularidad semanal insuficientemente robusta** | Si el mapeo a servicio o el ruido de calendario degradan demasiado la serie semanal | Se simplifica a granularidad mensual, más robusta frente a ruido, sacrificando resolución |

### Parte de la estrategia con mayor incertidumbre
El mapeo diagnóstico → servicio, porque depende de una regla construida por el equipo del proyecto y no de un campo nativo de la fuente.

### Alternativa global si el modelo no supera el baseline
El proyecto se replegaría a un panel puramente descriptivo de estacionalidad y tendencia histórica (sin componente predictivo), presentado igualmente en el dashboard, priorizando la honestidad del resultado sobre la ambición de la promesa inicial.
