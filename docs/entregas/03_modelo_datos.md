# Entrega 3 - Diseño del modelo de datos y capa gold del proyecto

> **Nota de revisión (respuesta al feedback recibido):** el feedback señala que varias tablas gold exigían granularidades que las fuentes públicas reales no garantizan (demanda semanal por servicio, episodios individuales, biomarcadores, reingreso a 30 días), que rellenar esos huecos con datos sintéticos permite demostrar el pipeline pero no sostener conclusiones clínicas ni un sistema real de apoyo a la decisión, y que los tres módulos seguían manteniendo un alcance demasiado alto. En consecuencia, este documento reduce el entregable a **un único MVP construido íntegramente sobre datos reales** (Módulo 1 - predicción de demanda), incorpora una fase explícita de **validación de campos y frecuencias reales en EMH y ESCRI** antes de cerrar el diseño de la capa gold, y reclasifica los Módulos 2 y 3 como **diseño conceptual no implementado**, condicionado a que esa validación confirme que existe una fuente real (no sintética) capaz de sostenerlos. La arquitectura de datos (capas, formato, principios de limpieza) se mantiene porque no es lo que estaba en cuestión; lo que se ajusta es la promesa de producto a la evidencia real disponible.

---

## 1. Resumen de la idea y datos del proyecto

### Problema que resuelve

Los reingresos evitables, las altas prematuras y las derivaciones reactivas entre centros son problemas costosos y estructurales del sistema sanitario español, agravados por la presión asistencial no anticipada. El proyecto completo (visión a largo plazo) aspira a abordar las tres caras del problema -cuánta demanda va a llegar, a qué centro debería derivarse un paciente cuando la red está saturada, y qué patologías/perfiles concentran el riesgo de reingreso- con evidencia real y agregada. **El TFM, sin embargo, no promete las tres piezas a la vez**: entrega una de ellas completa y validada sobre datos reales, y documenta el resto como diseño conceptual pendiente de validación de fuente.

### Por qué el MVP se centra en un único módulo

Elegir un único módulo responde directamente a dos problemas detectados en la revisión:

1. **Riesgo de granularidad no garantizada.** Prometer tres módulos obliga a asumir, sin haberlo comprobado, que EMH ofrece fecha de alta reconstruible a nivel semanal *y* desagregación por servicio, que ESCRI aporta contexto de camas útil, y que iCMBD desagrega el reingreso por patología y CCAA simultáneamente. Si alguno de esos supuestos no se confirma, la salida honesta es sintetizar el dato - y un dato sintético no puede sostener una conclusión clínica ni un sistema real de apoyo a la decisión, solo una demostración de pipeline.
2. **Alcance excesivo para el tiempo y los datos disponibles.** Tres módulos, cada uno con su propia validación de fuente, EDA, modelo y panel, diluyen el esfuerzo y aumentan la probabilidad de que ninguno quede completamente validado.

Por eso el criterio de selección del MVP es: **el módulo cuya fuente real ya está confirmada a la granularidad necesaria, sin depender de trámites de acceso especiales ni de indicadores cuya desagregación exacta está aún por confirmar.**

### Solución que se quiere construir

| Módulo | Qué resolvería | Granularidad | Fuente principal | Estado en este entregable |
|---|---|---|---|---|
| **1. Predicción de demanda asistencial** | Proyección de ingresos urgentes/programados, 4-8 semanas vista, por CCAA (y por servicio si la validación de la sección 2 lo confirma) | Semana × (servicio) × CCAA | EMH, ESCRI | **MVP — en desarrollo, datos reales** |
| **2. Índice de priorización de derivación** | Identificar qué combinaciones patología × CCAA concentran mayor presión estructural de derivación | Patología (CIE-10 2 dígitos) × CCAA × año | EMH, ESCRI, GRD | **Diseño conceptual — no implementado, pendiente de validar fuente real** |
| **3. Índice de riesgo de reingreso al alta** | Estratificar qué combinaciones patología × edad × CCAA concentran mayor riesgo de reingreso a 30 días | Patología × grupo de edad × CCAA × año | EMH, GRD, iCMBD | **Diseño conceptual — no implementado, pendiente de validar fuente real** |

El diseño de los Módulos 2 y 3 se conserva íntegro en la sección 10 (antes secciones 5.2/5.3) porque el trabajo de análisis de viabilidad ya hecho tiene valor y puede activarse si la validación de fuentes lo respalda - pero no forma parte de lo que este TFM promete entregar como sistema funcional.

### Fuentes de datos y tipo de información aportada (MVP)

| Fuente | Tipo de información | Uso en el proyecto |
|---|---|---|
| Encuesta de Morbilidad Hospitalaria (EMH) - INE, serie 2014-2023 | Altas con fecha real de alta, diagnóstico CIE-10, modalidad de ingreso, edad, sexo, CCAA | Fuente principal del MVP: reconstrucción de la serie semanal de ingresos |
| Estadística de Centros de Atención Especializada (antigua ESCRI) - Ministerio de Sanidad, serie 2014-2023 | Foto anual de camas instaladas/en funcionamiento, dotación tecnológica y personal por centro | Contexto estructural anual estático del MVP (capacidad de la red) |

GRD e iCMBD quedan documentadas como fuentes potenciales para una futura extensión (sección 10), pero no se usan en el MVP actual.

---

## 2. Validación de las fuentes: qué granularidad soportan realmente

Esta sección era, en la versión anterior, una descripción de la granularidad esperada. El feedback pide algo más estricto: **confirmar antes de avanzar, no asumir**, exactamente qué campos y frecuencias existen en EMH y ESCRI. Se separa en (2.1) el checklist de validación pendiente — la tarea prioritaria solicitada - y (2.2) lo ya documentado sobre periodicidad de publicación, que sigue siendo cierto pero no sustituye a la validación de campo.

### 2.1 Checklist de validación pendiente (acción prioritaria antes de cerrar el diseño gold)

Antes de dar por cerrada la capa gold del MVP, queda pendiente confirmar y documentar en el diario de proyecto:

- [ ] ¿El microdato de la EMH permite reconstruir la fecha real de alta con precisión suficiente para agregar a **semana epidemiológica**, o la granularidad temporal real disponible es más gruesa de lo asumido?
- [ ] ¿La EMH desagrega por **servicio hospitalario**, o esa desagregación no está garantizada y el MVP debe quedarse en semana × CCAA (sin servicio) hasta poder confirmarlo?
- [ ] ¿Qué campos de ESCRI están realmente disponibles a nivel de CCAA/año sin huecos relevantes (dotación de camas, personal), y cuáles tienen cobertura incompleta por centro o año?
- [ ] Confirmar el canal y el plazo real de acceso al microdato EMH (Área de Atención a Usuarios del INE): si el trámite se demora más de lo asumible para el calendario del TFM, documentar el impacto y la alternativa (agregados públicos ya disponibles vs. microdato solicitado).
- [ ] Si la desagregación por servicio no se confirma, documentar formalmente la reducción de alcance del MVP a semana × CCAA, dejando constancia de que no se generará ese campo de forma sintética para mantener la promesa original.

**Principio que gobierna esta validación:** si un campo o frecuencia no se puede confirmar contra la fuente real, el MVP se simplifica (menos desagregación, menos horizonte, menos módulos) antes que rellenarlo con datos sintéticos. Un dato inventado no es una granularidad menor, es una fuente distinta que no puede sostener las mismas conclusiones.

### 2.2 Periodicidad de publicación conocida (contexto, no sustituye a 2.1)

| Fuente | Periodicidad de publicación | Periodicidad real del dato | Consecuencia para el proyecto |
|---|---|---|---|
| **EMH (INE)** | Anual (12-18 meses de retraso) | Microdato con fecha de alta (precisión a confirmar, ver 2.1) | Serie semanal del MVP, condicionada a la confirmación del checklist |
| **ESCRI / SIAE (Sanidad)** | Anual | Foto a 31 de diciembre, no serie de ocupación | Contexto estructural anual estático, nunca disponibilidad de camas en tiempo real |

**Punto de atención ya conocido:** la página de microdatos de la EMH indica que, aunque los resultados agregados son de descarga libre, los ficheros de microdatos individuales solo se facilitan bajo condiciones especiales en el Área de Atención a Usuarios del INE, no por descarga directa sin trámite. Esto no invalida el MVP (si la desagregación semanal necesaria estuviera también en agregados públicos, se prioriza esa vía), pero condiciona el calendario si el microdato exacto es imprescindible; se documenta como riesgo en la sección 9.

---

## 3. Tecnología o formato de almacenamiento elegido

Se mantiene **Parquet** para la capa `processed` y **CSV** para `raw` y `gold`. No se utiliza base de datos relacional: la fuente del MVP es un fichero o consulta descargable sin actualización en streaming, y el volumen (miles de filas agregadas, no microdatos masivos) no exige transaccionalidad ni consultas concurrentes.

---

## 4. Estructura de capas de datos

```
data/
├── raw/          # Datos originales tal como se obtienen de cada fuente
├── processed/    # Datos limpios, tipados y parcialmente transformados
└── gold/         # Dataset final del MVP, listo para modelo y dashboard
```

| Capa | Contenido esperado | Granularidad |
|---|---|---|
| `raw/` | Microdatos/agregados EMH por año, ficheros anuales ESCRI | Una entrega anual por fuente |
| `processed/` | Ficheros Parquet limpios: tipos corregidos, nulos tratados, duplicados eliminados, `snake_case`, fechas ISO 8601 | EMH agregado a semana × CCAA (× servicio si se confirma en 2.1) |
| `gold/` | `gold_demanda_asistencial.csv` - único dataset del MVP | Ver sección 5 |

El diseño conceptual de Módulos 2 y 3 (sección 10) no genera ficheros en `gold/` mientras no se active; si en el futuro se valida su fuente, se añadirán como datasets adicionales sin modificar esta estructura.

---

## 5. Definición de la capa gold (MVP)

### 5.1 `gold_demanda_asistencial.csv`

**Granularidad:** semana epidemiológica × CCAA (× servicio, condicionado a la confirmación del checklist 2.1).
**Clave primaria:** `(anio, semana_epidemiologica, ccaa_codigo[, servicio])`.
**Campos principales:** `ingresos_urgentes` (objetivo), `ingresos_programados`, `ratio_urgentes_programados`, `indicador_estacional`, `festivo_semana`, `camas_totales_ccaa`, `tendencia_ingresos_4s`.
**Uso posterior:** forecasting SARIMA/Prophet, dashboard de demanda (Entregas 4 y 5).

**Nota de alcance:** si la validación de la sección 2.1 no confirma desagregación por servicio, este dataset se entrega a granularidad semana × CCAA únicamente, sin campo `servicio`, documentando el motivo. No se sintetiza ese nivel de detalle.

---

## 6. Diccionario de datos (MVP)

| Campo | Descripción | Tipo | Fuente | Obligatorio | Observaciones |
|---|---|---|---|---|---|
| `anio` | Año del registro | int | EMH | Sí | - |
| `semana_epidemiologica` | Semana ISO del año | int | EMH | Sí | - |
| `ccaa_codigo` | Código INE de CCAA | str | EMH | Sí | - |
| `servicio` | Servicio hospitalario | str | EMH | Condicionado | Solo si el checklist 2.1 confirma la desagregación real |
| `ingresos_urgentes` | Nº de ingresos urgentes de la semana | int | EMH | Sí | Variable objetivo |
| `ingresos_programados` | Nº de ingresos programados de la semana | int | EMH | Sí | - |
| `camas_totales_ccaa` | Capacidad estructural de la CCAA (contexto anual, no ocupación en tiempo real) | int | ESCRI | Sí | - |

El diccionario de campos de los Módulos 2 y 3 se mantiene en la sección 10, con la misma etiqueta de "diseño conceptual, no implementado".

---

## 7. Problemas de calidad esperados

- **Confirmación de granularidad real (EMH, ESCRI):** riesgo principal identificado en la revisión; se gestiona mediante el checklist de la sección 2.1 antes de cerrar el diseño, no durante la implementación.
- **Mapeo diagnóstico → servicio:** requiere reglas de mapeo documentadas que pueden introducir ambigüedad en códigos no clínicos, solo aplica si la desagregación por servicio se confirma.
- **Acceso a microdatos EMH:** sujeto a solicitud especial (ver sección 2.2); riesgo de calendario si el trámite se demora.
- **Ruptura estructural 2020-2021:** se excluye o se trata como variable exógena.

---

## 8. Decisiones de limpieza y transformación previstas

### Tratamiento de nulos
- Semanas sin registro se tratan como missing explícito, no como cero, para no distorsionar la estacionalidad.
- Si un campo del checklist 2.1 no se confirma, se elimina del diseño en lugar de imputarse o generarse sintéticamente.

### Variables derivadas a construir

| Variable derivada | Cálculo |
|---|---|
| `ratio_urgentes_programados` | `ingresos_urgentes / ingresos_programados` |
| `tendencia_ingresos_4s` | Media móvil de 4 semanas de `ingresos_urgentes` |

### Datos que se descartarán
- Años 2020-2021.
- Cualquier campo cuya granularidad real no se confirme en el checklist de la sección 2.1 (en vez de sustituirlo por una estimación o dato sintético).

---

## 9. Riesgos del modelo de datos

### ¿Qué parte está más clara?
Que EMH y ESCRI son fuentes reales, públicas y con serie histórica suficiente para un MVP de demanda; el principio de no usar datos sintéticos en ningún caso.

### ¿Qué parte genera más incertidumbre?
Exactamente la que señala la revisión: si la granularidad semanal por servicio está realmente garantizada por la EMH, o si el proyecto debe conformarse con semana × CCAA. Esta incertidumbre se resuelve en la fase de validación (sección 2.1), antes de construir el pipeline completo, no después.

### Alternativa si la desagregación por servicio no se confirma
El MVP se entrega a granularidad semana × CCAA, sin campo `servicio`, documentando la pérdida de resolución como decisión consciente y no como un defecto oculto.

### Alternativa si el acceso a microdatos EMH se demora
Priorizar los agregados públicos ya disponibles (si sostienen la granularidad necesaria) sobre la solicitud de microdato, documentando cualquier pérdida de resolución que eso implique.

---

## 10. Anexo — Diseño conceptual de Módulos 2 y 3 (no implementado en este entregable)

> Todo lo que sigue es el trabajo de diseño ya realizado sobre los Módulos 2 (presión de derivación) y 3 (riesgo de reingreso). Se conserva porque tiene valor de análisis, pero **no forma parte del MVP entregado**: no se construyen sus datasets gold, no se entrenan sus modelos y no aparecen en el dashboard de esta entrega. Solo se activarían si, en una futura fase, se confirma que GRD e iCMBD ofrecen la granularidad real necesaria - nunca completando esa granularidad con datos sintéticos.

### 10.1 Planteamiento

| Módulo | Qué resolvería | Granularidad | Fuente principal |
|---|---|---|---|
| **2. Índice de priorización de derivación** | Identificar qué combinaciones patología × CCAA concentran mayor presión estructural de derivación, para apoyar decisiones de planificación de red (no la derivación individual de un paciente en tiempo real) | Patología (CIE-10 2 dígitos) × CCAA × año | EMH, ESCRI, GRD |
| **3. Índice de riesgo de reingreso al alta** | Estratificar qué combinaciones patología × edad × CCAA concentran mayor riesgo de reingreso a 30 días, para apoyar la revisión de protocolos de alta en esos segmentos | Patología × grupo de edad × CCAA × año | EMH, GRD, iCMBD |

Ninguno de los dos produciría una recomendación para un paciente concreto en un momento concreto (eso exigiría ocupación de camas en tiempo real y trazabilidad individual entre ingresos, que ninguna fuente pública ofrece). Producirían, en cambio, un índice de riesgo/priorización por segmento (patología × CCAA × edad).

### 10.2 Fuentes adicionales que exigirían

| Fuente | Tipo de información | Uso previsto |
|---|---|---|
| Grupos Relacionados por el Diagnóstico (GRD) - SNS | Estancia media y coste de referencia por diagnóstico | Variable de desviación de estancia (severidad/complejidad) |
| iCMBD (indicadores y ejes de análisis del CMBD) - Ministerio de Sanidad | Indicadores oficiales agregados: tasa de reingresos, mortalidad, estancia media, por CCAA/hospital/diagnóstico | Variable objetivo real de M3, si su desagregación llega a patología × CCAA |

**Punto de atención abierto, no resuelto:** falta confirmar si iCMBD desagrega la tasa de reingreso a nivel de patología × CCAA simultáneamente, o solo por uno de los dos ejes, y si el acceso es de consulta interactiva o de descarga masiva. Mientras esto no se confirme, M3 no pasa de diseño conceptual.

### 10.3 Datasets gold propuestos (conceptuales)

**`gold_presion_derivacion.csv` (M2)** — clave `(anio, ccaa_codigo, patologia_cie10)`:

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
| `camas_totales_ccaa` | int | Capacidad estructural de la CCAA — **contexto anual, nunca disponibilidad real de camas para decidir una derivación** | ESCRI |
| `score_presion_derivacion` | float | Variable de salida: índice combinado | Derivado (modelo) |

**`gold_riesgo_reingreso.csv` (M3)** — clave `(anio, ccaa_codigo, patologia_cie10, grupo_edad)`:

| Campo | Tipo | Descripción | Fuente |
|---|---|---|---|
| `anio` | int | Año del registro | EMH / iCMBD |
| `ccaa_codigo` | str | Código INE de CCAA | EMH |
| `patologia_cie10` | str | Grupo diagnóstico | EMH |
| `grupo_edad` | str | Franja decenal | EMH |
| `altas_totales` | int | Nº de altas del segmento | EMH |
| `tasa_reingreso_30d` | float | Variable objetivo: tasa oficial de reingreso a 30 días - **debe proceder de un indicador ya agregado y real (iCMBD); si esa granularidad no se confirma, el módulo no se sostiene con datos sintéticos** | iCMBD |
| `estancia_media_dias` | float | Estancia media real del segmento | EMH |
| `desviacion_estancia` | float | Estancia real vs. GRD esperada | GRD (derivado) |
| `mortalidad_intrahosp_pct` | float | % de altas por fallecimiento del segmento | EMH |
| `pct_ingreso_urgente` | float | % de ingresos de carácter urgente en el segmento | EMH |
| `camas_totales_ccaa` | int | Contexto estructural anual | ESCRI |
| `score_riesgo_reingreso` | float | Variable de salida: riesgo predicho/explicado por el modelo | Derivado (modelo) |

### 10.4 Por qué se descarta la versión en tiempo real / individual (si algún día se activan)

| Elemento que exigiría un sistema en tiempo real/individual | Por qué no es viable con fuentes públicas | Solución que se adoptaría |
|---|---|---|
| Ocupación de camas por centro, actualizada varias veces al día | ESCRI es una foto anual, no una serie de ocupación | Capacidad estructural anual como contexto, nunca como variable de disponibilidad en tiempo real |
| Identificador de paciente para vincular ingreso y reingreso | La EMH tiene como unidad estadística el alta, no el paciente; no hay trazabilidad pública entre altas de la misma persona | Usar `tasa_reingreso_30d` de iCMBD, ya calculada por el Ministerio con la trazabilidad interna que el público no tiene; si en algún momento se explorase granularidad de episodio individual, sería imprescindible contar con un identificador de paciente real, no un `episodio_id` |
| Biomarcadores y constantes vitales individuales al momento del alta | No publicados a nivel de paciente en ninguna fuente pública | No se usan; se sustituirían por proxies agregados de severidad (mortalidad, desviación de estancia) a nivel de segmento |

### 10.5 Condición para activar este diseño

Los Módulos 2 y 3 solo pasarían de "diseño conceptual" a "en desarrollo" si: (a) se confirma que GRD e iCMBD ofrecen la granularidad real necesaria, siguiendo el mismo tipo de checklist de la sección 2.1; y (b) el MVP (Módulo 1) ya está entregado y validado, para no repetir el problema de alcance señalado en la revisión. En ningún caso se completaría un hueco de granularidad con datos sintéticos para forzar su activación.
