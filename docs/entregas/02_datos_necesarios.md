# Entrega 2 - Selección de idea de proyecto y análisis de datos necesarios

---

## 1. Idea seleccionada
**Sistema de scoring inteligente para la evaluación de criterios de alta y derivación hospitalaria**
### Párrafo 1 - Problema que resuelve
Las altas prematuras y los reingresos evitables constituyen uno de los problemas más costosos y persistentes del Sistema Nacional de Salud. Cuando un hospital opera bajo presión asistencial elevada, la decisión de dar el alta a un paciente se ve condicionada por la necesidad urgente de liberar camas, lo que puede derivar en reingresos innecesarios con peor pronóstico clínico y mayor coste para el sistema. Según datos del INE 2023, entre un 20-25% de los reingresos a 30 días en patologías crónicas de alta prevalencia como la insuficiencia cardíaca son potencialmente evitables, lo que representa un ahorro estimado superior a los 55 millones de euros anuales solo en esa categoría diagnóstica. De forma paralela, la derivación de pacientes entre centros hospitalarios se realiza frecuentemente de manera reactiva: en 2023 se registraron 283.792 traslados no programados, un 5,83% del total de altas, muchos de ellos en condiciones de urgencia que elevan el riesgo clínico del paciente y duplican los costes operativos. Ambos problemas comparten una raíz común: la ausencia de herramientas basadas en datos que apoyen estas decisiones de forma objetiva, homogénea y anticipada.
### Párrafo 2 - Solución planteada
El proyecto propone desarrollar un sistema de apoyo a la decisión clínica basado en técnicas de Data Science con dos componentes principales. El primero es un modelo de clasificación supervisada que estime el riesgo de reingreso a 30 días de cada paciente en el momento previo al alta, en función de su perfil clínico, diagnóstico principal, comorbilidades, constantes vitales y variables operativas del episodio. El segundo es un algoritmo de derivación hospitalaria multicriterio que, ante situaciones de alta ocupación, identifique el centro receptor más adecuado de la red sanitaria evaluando disponibilidad de camas especializadas, compatibilidad tecnológica, tiempo de traslado estimado y perfil diagnóstico del paciente. Ambos componentes se alimentan de datos públicos anonimizados del SNS y de un dataset sintético generado a partir de las distribuciones estadísticas observadas en las fuentes oficiales, siguiendo una metodología documentada y reproducible. El sistema no busca sustituir el criterio clínico del profesional sanitario, sino ofrecer una capa de apoyo basada en datos que permita decisiones más fundamentadas, homogéneas y seguras.
### Párrafo 3 - MVP del proyecto final
El producto mínimo viable que se presentará al final del curso consistirá en un dashboard interactivo con tres elementos funcionales integrados. En primer lugar, un análisis exploratorio y visual de los datos históricos del SNS (2014-2023), con segmentación por patología, comunidad autónoma y evolución temporal de indicadores clave como ocupación, reingresos y traslados. En segundo lugar, un modelo de clasificación XGBoost entrenado sobre un dataset sintético que asigne a cada paciente simulado un nivel de riesgo de reingreso (alto, moderado, bajo), acompañado de explicabilidad mediante SHAP para identificar los factores clínicos más determinantes en cada caso. En tercer lugar, una simulación del módulo de derivación inteligente que, dado un escenario de ocupación hospitalaria, calcule y visualice la puntuación de cada centro candidato de la red y proponga la derivación óptima. El MVP se construirá íntegramente con datos públicos y sintéticos, sin acceso a datos clínicos individuales identificables.

---

## 2. Datos necesarios
### Variables o campos necesarios
El proyecto requiere datos organizados en torno a cuatro categorías, coherentes con las identificadas en la literatura clínica de referencia sobre predicción de reingreso y derivación hospitalaria.
**Variables demográficas y administrativas:** grupo de edad en franjas decenales, sexo, comunidad autónoma de residencia, tipo de ingreso (urgente o programado), servicio de ingreso, GRD asignado y semana epidemiológica del ingreso.
**Variables clínicas del episodio:** diagnóstico principal codificado en CIE-10, diagnósticos secundarios y comorbilidades relevantes (índice de Charlson como variable derivada), procedimientos realizados durante el ingreso, y resultados de biomarcadores analíticos clave según patología (troponina en cardiopatía isquémica, PCT y lactato en sepsis, BNP en insuficiencia cardíaca, PCR en procesos infecciosos).
**Variables de proceso asistencial:** duración de la estancia en días, diferencia entre estancia real y estancia media del GRD correspondiente, número de ingresos previos en los últimos 12 meses, tiempo transcurrido desde el último ingreso, y pendencias diagnósticas en el momento del alta.
**Variables operativas y de red:** tasa de ocupación del servicio en el momento del alta o del ingreso, nivel de presión de urgencias en las últimas 24 horas, ocupación de los centros de la red del área sanitaria, y tendencia de demanda proyectada en horizonte de 24 y 72 horas.

### Nivel de granularidad
Para el análisis exploratorio, la granularidad adecuada es a nivel de patología (CIE-10 a dos dígitos), desagregada por comunidad autónoma y con periodicidad anual. Para el modelo de clasificación supervisada, se trabajará con registros individuales del dataset sintético generado a partir de las distribuciones estadísticas de las fuentes públicas, con una observación por episodio de hospitalización.

### Profundidad histórica necesaria
Una serie temporal de 10 años (2014-2023) es necesaria para capturar patrones estacionales, tendencias de demanda por patología y el impacto de eventos extraordinarios como la pandemia. El período 2020-2021 se tratará como variable exógena o se excluirá del entrenamiento según el análisis exploratorio inicial, dado su carácter atípico.

### Volumen aproximado de datos
Los datasets públicos del INE y del Ministerio de Sanidad contienen varios miles de registros agregados por patología, comunidad autónoma y año, suficientes para el análisis exploratorio. Para el modelo supervisado se generará un dataset sintético de entre 50.000 y 100.000 registros simulados a partir de las distribuciones estadísticas observadas, volumen suficiente para entrenar y validar modelos de clasificación con garantías estadísticas.

### Datos imprescindibles vs. deseables
**Imprescindibles:** diagnóstico principal por CIE-10, altas por patología y modalidad de ingreso (urgente/programado), estancia media por GRD, número de reingresos por patología, número de traslados entre centros, indicadores de ocupación hospitalaria por servicio y período, y grupo de edad y sexo como variables de segmentación.
**Deseables pero no imprescindibles:** constantes vitales individuales, resultados de biomarcadores analíticos y pendencias diagnósticas al alta. Estas variables, no disponibles en las fuentes públicas a nivel individual, se incorporarán al dataset sintético a partir de las distribuciones estadísticas documentadas en la literatura clínica de referencia.

---

## 3. Fuentes de datos previstas

### Fuente principal 1: Encuesta de Morbilidad Hospitalaria (EMH) - INE

- **URL:** https://www.ine.es/dyngs/INEbase/es/operacion.htm?c=Estadistica_C&cid=1254736176778&menu=resultados
- **Tipo de acceso:** Abierto y público, sin restricciones de descarga.
- **Formato:** CSV y Excel descargables directamente.
- **Histórico disponible:** Serie completa 2014-2023.
- **Estabilidad:** Fuente oficial del INE, publicación anual con alta estabilidad y mantenimiento garantizado.
- **Contenido principal:** Altas hospitalarias por diagnóstico principal (CIE-10), modalidad de ingreso (urgente/programado), estancia media, reingresos, traslados entre centros, mortalidad intrahospitalaria, desagregado por edad, sexo y comunidad autónoma. Esta fuente ya ha sido trabajada y procesada previamente, por lo que se dispone de los datos descargados y transformados.

### Fuente principal 2: Estadística de Establecimientos Sanitarios con Régimen de Internado (ESCRI) - Ministerio de Sanidad
- **URL:** https://www.sanidad.gob.es/estadEstudios/estadisticas/estHospiInternado/inforRecopilaciones/home.htm
- **Tipo de acceso:** Abierto y público.
- **Formato:** Excel.
- **Histórico disponible:** Serie 2014-2023.
- **Estabilidad:** Fuente oficial del Ministerio de Sanidad, alta estabilidad.
- **Contenido relevante:** Indicadores de ocupación hospitalaria, índice de rotación, presión de urgencias, dotación tecnológica por centro y comunidad autónoma. Permite construir las variables operativas de red necesarias para el módulo de derivación. Esta fuente también ha sido trabajada y procesada previamente.

### Fuente complementaria: Grupos Relacionados por el Diagnóstico (GRD) - Ministerio de Sanidad
- **URL:** https://www.sanidad.gob.es/estadEstudios/estadisticas/cmbd.do
- **Tipo de acceso:** Abierto y público.
- **Formato:** Excel y CSV.
- **Contenido relevante:** Estancia media esperada por GRD, coste medio por episodio. Permite construir la variable de desviación entre estancia real y estancia media esperada, uno de los predictores más relevantes del riesgo de alta prematura identificados en la literatura clínica.

### Riesgos detectados
El principal riesgo es la **granularidad limitada** de los datos públicos: las fuentes disponibles ofrecen datos agregados por patología, comunidad autónoma y año, pero no registros individuales de pacientes. Esto impide entrenar directamente el modelo de clasificación supervisada con datos reales. La estrategia de mitigación es la **generación de un dataset sintético** construido a partir de las distribuciones estadísticas observadas en las fuentes públicas, técnica ampliamente utilizada y validada en la literatura de machine learning aplicado a salud.
Un segundo riesgo es la **discontinuidad en la serie temporal** producida por el período 2020-2021, que se gestionará como variable exógena o mediante exclusión controlada del período atípico.
Un tercer riesgo menor es la **heterogeneidad entre comunidades autónomas** en los criterios de codificación y reporte, que puede requerir un proceso de normalización más intensivo de lo esperado en la fase de preparación de datos.

---

## 4. Consideraciones de privacidad y protección de datos
El proyecto trabaja exclusivamente con datos públicos agregados y anonimizados en origen, publicados por organismos oficiales (INE y Ministerio de Sanidad). Ninguna de las fuentes previstas contiene información personal identificable: los datos de la EMH y la ESCRI se publican a nivel de patología, comunidad autónoma y período, sin ningún atributo que permita identificar a pacientes individuales.
En lo que respecta al dataset sintético generado para el entrenamiento del modelo supervisado, se trata de datos completamente artificiales construidos a partir de distribuciones estadísticas documentadas, por lo que no existe riesgo de reidentificación ni de tratamiento de datos personales en ningún caso.
El proyecto no requiere acceso a historiales clínicos, sistemas hospitalarios de información (HIS/LIS), datos de pacientes concretos ni ningún tipo de información sujeta al Reglamento General de Protección de Datos (RGPD) o a la normativa sanitaria específica de protección de datos clínicos (Ley 41/2002 de autonomía del paciente). Por tanto, no es necesario establecer protocolos de anonimización, obtener consentimientos ni solicitar permisos especiales.
Desde el punto de vista ético, el modelo de scoring de riesgo de reingreso se evaluará explícitamente en subgrupos definidos por edad, sexo y comunidad autónoma para detectar y corregir posibles sesgos sistemáticos en variables protegidas, como parte del proceso de validación del modelo.

---

## 5. Viabilidad inicial del proyecto
**¿Es viable obtener los datos necesarios?** Sí, con alta seguridad. Las dos fuentes principales son públicas, estables, accesibles sin restricciones y han sido ya descargadas y procesadas previamente. No existe ninguna barrera de acceso, pago o permiso que pueda comprometer la obtención de los datos.
**¿La información tiene suficiente calidad, granularidad y profundidad histórica?** Suficiente para el análisis exploratorio y el modelado a nivel agregado. La granularidad individual necesaria para el modelo supervisado se resuelve mediante generación de datos sintéticos, una estrategia metodológicamente sólida y apropiada para un proyecto académico de estas características, con precedente directo en la literatura de machine learning aplicado a salud.
**¿La idea puede desarrollarse de forma realista durante el curso?** Sí. El alcance está bien acotado en tres componentes concretos: análisis exploratorio de datos públicos del SNS 2014-2023, construcción y entrenamiento del modelo de clasificación XGBoost con explicabilidad SHAP, y simulación del algoritmo de derivación multicriterio. Estas tareas son completamente abordables con las herramientas y el tiempo disponibles en el curso.
**¿Qué parte del proyecto es más arriesgada?** La construcción del dataset sintético con suficiente realismo clínico para que el modelo de clasificación sea representativo. Para mitigar este riesgo, el diseño de las distribuciones se apoyará en los patrones estadísticos observados en la EMH del INE y en la literatura clínica de referencia sobre predicción de reingreso hospitalario, siguiendo una metodología documentada y reproducible.
**¿Qué alternativa existe si la fuente principal falla?** Si la EMH o la ESCRI presentaran problemas inesperados de acceso o calidad, la alternativa inmediata es utilizar los datasets de actividad hospitalaria del NHS (Reino Unido), disponibles en formato abierto en NHS Digital con estructura similar y mayor granularidad individual, que podrían emplearse como fuente de entrenamiento alternativa manteniendo el mismo enfoque metodológico.
