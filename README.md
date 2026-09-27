# Índice de Presión Asistencial del SNS

**Trabajo Fin de Máster: Máster en Data Science & AI (Evolve)**
Autora: Irene Rodríguez Sánchez · Curso 2025-2026

Sistema de apoyo a la planificación hospitalaria basado en datos reales del Sistema Nacional de Salud (SNS): un índice estructural de presión asistencial por patología y comunidad autónoma, construido y validado sobre datos públicos abiertos del INE y el Ministerio de Sanidad.

**Demo interactiva:** https://claude.ai/artifact/LYuL3HSnTFjuB7r7tqY6tc

---

## El problema

Las direcciones médicas de área y la gestión de camas del SNS comparan hoy la presión asistencial entre comunidades autónomas y patologías de forma reactiva, sin un panel único basado en datos reales. Este proyecto valida ocho fuentes públicas del INE y el Ministerio de Sanidad y confirma un hallazgo que determina todo su diseño: **ninguna cruza simultáneamente diagnóstico, territorio y tiempo**. A partir de esa limitación, el alcance se acota a un único MVP: un índice estructural de presión asistencial por patología × comunidad autónoma para 2023, complementado con un bloque descriptivo de tendencia nacional 2014-2023.

El proyecto sigue la metodología **CRISP-DM**, con especial énfasis en la honestidad metodológica: cada limitación de las fuentes se documenta como resultado, no se disimula, y la evaluación final se apoya en una validación cruzada por grupos (no en un K-Fold simple) precisamente para evitar sobreestimar lo que el modelo demuestra.

## Resultado principal

| | K-Fold simple | GroupKFold por patología | **GroupKFold por CCAA** |
|---|---|---|---|
| Mejora del modelo sobre el baseline | +8,6% | +5,4% (R² negativo) | **+0,7%** |

El índice separa **patologías** con solidez (dominadas por su perfil clínico nacional), pero la validación correcta -forzando al modelo a generalizar a comunidades autónomas nunca vistas- muestra que la contribución **territorial** es real pero mucho más modesta de lo que sugería una validación ingenua. Esta corrección, no la cifra optimista de partida, es el resultado central de la evaluación. Detalle completo en [`docs/04_05_modelado_evaluacion.md`](docs/04_05_modelado_evaluacion.md).

## Estructura del repositorio

```
.
├── README.md
├── docs/                              # Texto completo del TFM, por sección
│   ├── 00_resumen.md
│   ├── 03_preparacion_datos.md
│   ├── 04_05_modelado_evaluacion.md
│   ├── 06_07_despliegue_conclusiones.md
│   ├── Anexo_A_diccionario_datos.md
│   └── Anexo_B_C.md
├── notebooks/                         # Google Colab, ejecutables de principio a fin
│   ├── tfm_03_preparacion_datos.ipynb
│   ├── tfm_04_modelado.ipynb
│   └── tfm_05_evaluacion.ipynb
├── src/
│   └── tfm_pipeline_seccion3.py       # Script standalone equivalente al notebook 3
├── data/
│   ├── raw/                           # Los 8 .xlsx originales (INE / Sanidad)
│   └── processed/                     # Capa gold generada por el pipeline
│       ├── gold_presion_asistencial_2023.csv
│       ├── gold_contexto_ccaa_2023.csv
│       └── diccionario_datos_gold_2023.csv
├── demo/
│   └── tfm_demo_dashboard.html        # Dashboard interactivo, autocontenido
└── TFM_Evolve_Irene.pdf               # Documento final completo
```

> Los notebooks y el script asumen que los 8 `.xlsx` están en el mismo directorio de ejecución (o se suben manualmente en la primera celda si se ejecuta en Colab).

## Fuentes de datos

Ocho ficheros públicos y abiertos, sin restricciones de acceso, correspondientes al ejercicio 2023 salvo dos series históricas 2014-2023:

| Fichero | Organismo | Contenido |
|---|---|---|
| `Alta_Diagnostico_Provincia.xlsx` | INE - Encuesta de Morbilidad Hospitalaria | Altas por diagnóstico × CCAA/provincia |
| `Alta_Motivo_ingreso.xlsx` | INE - EMH | Traslado y fallecimiento por diagnóstico (nacional) |
| `Alta_Urgencia_ingreso.xlsx` | INE - EMH | % ingreso urgente por diagnóstico (nacional) |
| `Estancia_media.xlsx` | INE - EMH | Estancia media por diagnóstico (nacional) |
| `Altas_Diagnostico_principal.xlsx` | INE - EMH | Altas por diagnóstico × edad (no usado en la capa gold) |
| `Tablas_CCAA_2023.xlsx` | Ministerio de Sanidad - ESCRI | Hospitales, camas, personal, urgencias por CCAA |
| `Actividad_Evolucion_2014-2023.xlsx` | Ministerio de Sanidad - ESCRI | Serie nacional de actividad hospitalaria |
| `Dotacion_Evolucion_2014-2023.xlsx` | Ministerio de Sanidad - ESCRI | Serie nacional de dotación de camas |

Ficha técnica completa de cada fuente en [`docs/Anexo_B_C.md`](docs/Anexo_B_C.md).

## Cómo reproducirlo

**Opción A — Google Colab (recomendado):**
1. Abre `notebooks/tfm_03_preparacion_datos.ipynb` en Colab.
2. Ejecuta la primera celda y sube los 8 `.xlsx` cuando lo pida.
3. Ejecuta el resto del notebook; genera los 3 CSV de la capa gold.
4. Repite con `tfm_04_modelado.ipynb` y `tfm_05_evaluacion.ipynb`, subiendo `gold_presion_asistencial_2023.csv` cuando lo pidan.

**Opción B — local:**
```bash
pip install pandas openpyxl scikit-learn shap matplotlib
python src/tfm_pipeline_seccion3.py   # genera la capa gold en data/processed/
```

## La demo

`demo/tfm_demo_dashboard.html` es una página autocontenida (sin backend, sin dependencias de red salvo tipografía y Chart.js vía CDN) que consume directamente el índice ya calculado. Dos vistas:
- **Índice de presión 2023**: buscador por comunidad autónoma + patología, ranking territorial, top 10 combinaciones de mayor presión.
- **Tendencia nacional 2014-2023**: KPIs y gráficos de la serie histórica.

Ábrela directamente en cualquier navegador, sin instalación.

## Principios metodológicos

- **Sin datos sintéticos.** Todo el proyecto se construye y valida con fuentes 100% reales y públicas; los módulos que exigirían datos sintéticos (riesgo de reingreso, derivación multianual) se descartan y quedan documentados como diseño conceptual no implementado ([Anexo C](docs/Anexo_B_C.md)).
- **Limitación de fuente como resultado, no como nota al margen.** La imposibilidad de cruzar diagnóstico + territorio + tiempo en una sola fuente determina el alcance del proyecto desde la fase de comprensión de datos.
- **Validación honesta sobre validación favorable.** La evaluación final usa GroupKFold por CCAA precisamente porque es el esquema que puede desmentir el resultado más optimista, no el que mejor lo confirma.

## Limitaciones principales

- El índice no generaliza con solidez a comunidades autónomas nunca vistas (+0,7% de mejora bajo la validación correcta).
- El bloque de tendencia nacional es descriptivo: 10 puntos anuales no permiten forecasting con garantías estadísticas.
- No hay relación causal establecida entre las variables estructurales y la presión asistencial, solo asociación.

Detalle completo en [`docs/06_07_despliegue_conclusiones.md`](docs/06_07_despliegue_conclusiones.md), sección 7.

## Documento completo

El TFM completo (0. Resumen a 7. Conclusiones, más anexos) está en [`TFM_Evolve_Irene.pdf`](TFM_Evolve_Irene.pdf).
