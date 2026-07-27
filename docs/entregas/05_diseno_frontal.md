# Entrega 5 - Diseño del frontal y experiencia de usuario del producto

## 1. Resumen de la solución y del usuario

El proyecto predice la demanda asistencial hospitalaria (ingresos urgentes y programados) a 4-8 semanas vista, por servicio y comunidad autónoma, a partir de la serie histórica 2014-2023 de la Encuesta de Morbilidad Hospitalaria (INE) y la Estadística de Establecimientos Sanitarios (Ministerio de Sanidad).

**Usuario principal:** dirección médica / gestión de camas de un hospital o área sanitaria. Es un perfil no técnico que necesita apoyar decisiones de dotación de personal y camas, no interpretar un modelo estadístico.

**Necesidad concreta:** decidir, con 4-8 semanas de antelación, si hace falta reforzar la dotación de un servicio determinado antes de que llegue el pico de temporada, en lugar de reaccionar cuando la presión asistencial ya es alta.

**Tipo de producto:** dashboard analítico con un bloque de exploración histórica (EDA) y un bloque de forecasting. No es un clasificador ni un recomendador de acciones individuales: expone una proyección con su incertidumbre y dejar la decisión de dotación en manos del gestor.

**Resultado principal del frontal:** una proyección semanal de ingresos por servicio y CCAA, con intervalo de confianza, comparada contra el error del baseline, y una tabla exportable de las próximas semanas para trasladar a la planificación de turnos.

## 2. Imagen mockup del frontal

![Mockup del frontal](../assets/05_mockup_frontal.png)

La pantalla principal muestra, de arriba abajo: los filtros de CCAA, servicio y horizonte; un aviso permanente sobre el desfase de la fuente de datos; el gráfico de histórico y proyección con banda de confianza; los indicadores clave (predicción puntual, mejora frente al baseline, confianza del modelo); un panel de estacionalidad por servicio; una tabla exportable de las próximas semanas; y las acciones de exportación.

## 3. Justificación del diseño

### 3.1. Utilidad y valor de la solución

El frontal resuelve una tarea muy concreta: decidir si conviene anticipar dotación de camas o personal en un servicio antes de que empiece a subir la presión asistencial. Lo hace mostrando la proyección directamente superpuesta al histórico reciente, para que el gestor vea de un vistazo si la subida prevista es la habitual de temporada o si se sale del patrón.

Se ha decidido mostrar siempre, sin necesidad de profundizar, tres cosas: el número previsto, el rango de incertidumbre y la mejora del modelo frente a un baseline naive. Son los tres datos mínimos para tomar una decisión de dotación con criterio: cuánto, con qué margen de error, y si merece la pena confiar en la proyección más que en "lo que pasó el año pasado por estas fechas".

Se ha decidido **no** mostrar en la pantalla principal: los parámetros del modelo (SARIMA/Prophet), el detalle del proceso de entrenamiento, ni los tests estadísticos (Dickey-Fuller, ACF/PACF) que sí forman parte del EDA técnico de la Entrega 4. Esa información es imprescindible para el equipo del proyecto pero no aporta nada a la decisión del gestor; sobrecargarla habría diluido el mensaje principal. Queda disponible en el detalle técnico del panel de estacionalidad para quien quiera profundizar.

El resultado analítico se convierte en acción a través de la tabla de próximas semanas: es exportable en CSV para que el gestor pueda trasladarla directamente a la herramienta de planificación de turnos que ya use el hospital, sin tener que volver a teclear los números del dashboard.

### 3.2. Flujo de usuario

1. **Punto de entrada:** el gestor accede al dashboard y ve de inmediato el aviso de frescura de datos (última EMH disponible, fecha de generación del panel). Esto fija expectativas antes de mirar ningún número: el sistema es de planificación estructural, no una alerta en tiempo real.
2. **Entradas o selecciones:** el gestor elige CCAA, servicio y horizonte (2/4/8 semanas) mediante los filtros superiores. No hay más parámetros que introducir: el sistema no pide datos clínicos ni operativos al usuario.
3. **Procesamiento:** en segundo plano se ejecuta el modelo de forecasting (SARIMA/Prophet) sobre `gold_demanda_asistencial.csv`, sin que el gestor necesite saber qué modelo es ni cómo se entrenó.
4. **Resultado:** el gráfico principal, los indicadores clave y la tabla de próximas semanas. El gestor sabe si confiar en la proyección mirando dos señales: el ancho de la banda de confianza (a más ancha, más incertidumbre) y la comparación explícita "mejora vs. baseline", que le dice si el modelo aporta algo por encima de mirar el mismo dato del año pasado.
5. **Acción:** exportar la tabla (CSV) para repartir turnos y dotación, o descargar el informe (PDF) para llevarlo a una reunión de dirección. El dashboard no ejecuta ninguna acción operativa por sí mismo (no reserva camas ni asigna personal): es una herramienta de apoyo a la decisión, no un sistema operativo automatizado.
6. **Excepciones:** cuando una combinación CCAA/servicio tiene serie histórica insuficiente (por ejemplo, Illes Balears/Pediatría en el mockup), la fila correspondiente se marca como "cobertura baja" y se oculta el intervalo de confianza en lugar de mostrar un número poco fiable sin advertencia. Si el modelo no llega al umbral de mejora del 15-20% frente al baseline para una combinación concreta (criterio definido en la Entrega 4), el panel se repliega a mostrar solo el patrón estacional histórico para esa combinación, sin componente predictivo, en vez de forzar una proyección poco fiable.

### 3.3. Experiencia de usuario

**Jerarquía visual:** lo primero que debe captar la atención es el gráfico de histórico y proyección, porque es la vista que responde a la pregunta central ("¿qué va a pasar y con qué margen de error?"). Los indicadores clave están a su derecha, en segundo plano visual pero siempre visibles sin hacer scroll. El detalle técnico (estacionalidad, tabla completa) queda debajo, para quien quiera profundizar.

**Simplicidad:** se ha evitado convertir el dashboard en un panel de control estadístico. No aparecen métricas de ajuste del modelo (AIC, RMSE por parámetro, residuos) en la pantalla principal; solo la métrica que el gestor puede interpretar sin formación técnica (MAE expresado como "% de mejora frente al baseline").

**Legibilidad y consistencia:** una única paleta de estado se repite en todo el panel — verde para "fiable/dentro de lo esperado", ámbar para "atención/incertidumbre", rojo-ladrillo para "aviso de cobertura o desviación". El mismo código de color se usa en la tabla, en las tarjetas de indicadores y en la línea de "hoy" del gráfico, para que el gestor no tenga que reaprender el significado de un color en cada bloque.

**Contexto y confianza:** ninguna predicción se muestra como un número aislado. Siempre va acompañada de su intervalo de confianza, de la comparación contra el baseline y, en el panel de estacionalidad, del patrón histórico de referencia para juzgar si la proyección actual es coherente con años anteriores.

**Control del usuario:** el gestor puede cambiar CCAA, servicio y horizonte libremente y explorar distintas combinaciones antes de exportar nada; el sistema no toma ninguna decisión de forma autónoma, solo la presenta para que el gestor la revise y decida.

**Feedback del sistema:** el aviso de frescura de datos, el estado "cobertura baja" en la tabla y la tarjeta de aviso de cobertura son las tres formas de feedback explícito del panel. Comunican en el lenguaje del propio panel (no técnico) qué limitación tiene el dato antes de que el gestor confíe en él.

## 4. Presentación de resultados y explicabilidad

**Resultado principal:** una predicción puntual de ingresos urgentes por semana, servicio y CCAA, siempre acompañada de su intervalo de confianza al 95%.

**Información adicional para interpretarlo:** la serie histórica reciente superpuesta a la proyección, el patrón de estacionalidad de referencia por servicio, y la comparación explícita de error frente al baseline naive (definida como criterio de aceptación en la Entrega 4).

**Cómo se evita presentar la estimación como una certeza:** el intervalo de confianza se dibuja siempre junto al valor puntual, nunca se muestra un número suelto; y cuando la cobertura de datos de una combinación CCAA/servicio es insuficiente, se retira el intervalo y se marca el estado en vez de aparentar una precisión que no existe.

**Información técnica reservada a vista de detalle:** los tests de estacionariedad, el orden del modelo SARIMA, la descomposición tendencia/estacionalidad/residuo y el desglose de error por horizonte (2/4/8 semanas) de la Entrega 4 no aparecen en la pantalla principal; quedarían en un enlace "ver detalle" desde la tabla, pensado para el propio equipo del proyecto o para un perfil analítico, no para el gestor de camas.

**IA generativa:** no se utiliza IA generativa en el MVP. El panel se apoya exclusivamente en resultados controlados del modelo de forecasting (predicción, intervalo, comparación con baseline); no hay narrativa generada automáticamente que pueda inventar causas o descontextualizar la incertidumbre del modelo.

## 5. Alcance del MVP

**Se implementará y será funcional:** el bloque de EDA (estacionalidad, tendencia por servicio/CCAA) y el bloque de forecasting con SARIMA/Prophet sobre `gold_demanda_asistencial.csv`, los filtros de CCAA/servicio/horizonte, el gráfico de histórico y proyección con intervalo de confianza, los indicadores clave, y la exportación de la tabla en CSV.

**Será únicamente representación visual en el mockup, sin implementación real:** la descarga de informe en PDF (se documenta como funcionalidad deseable, no crítica para el MVP) y el enlace "ver detalle" por fila de la tabla hacia una vista técnica ampliada.

**Tecnología prevista:** Streamlit o Dash como framework de frontal, con Plotly para los gráficos interactivos (histórico, proyección, banda de confianza y small multiples de estacionalidad). No se prevé backend ni base de datos propia: el dashboard lee directamente el CSV generado por el pipeline de la Entrega 3/4 y recalcula la proyección al publicarse cada nueva edición de la EMH.
