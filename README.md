# Índice de Presión Asistencial del SNS

**Trabajo Fin de Máster: Máster en Data Science & AI (Evolve)**
Autora: Irene Rodríguez Sánchez · Curso 2025-2026

Sistema de apoyo a la planificación hospitalaria basado en datos reales del Sistema Nacional de Salud (SNS): un índice estructural de presión asistencial por patología y comunidad autónoma, construido y validado sobre datos públicos abiertos del INE y el Ministerio de Sanidad, dirigido a personal sanitario (dirección médica, jefaturas de servicio y gestión de camas).

---

## El problema

Las direcciones médicas de área y la gestión de camas del SNS comparan hoy la presión asistencial entre comunidades autónomas y patologías de forma reactiva, sin un panel único basado en datos reales. Este proyecto valida ocho fuentes públicas del INE y el Ministerio de Sanidad y confirma un hallazgo que determina todo su diseño: **ninguna cruza simultáneamente diagnóstico, territorio y tiempo**. A partir de esa limitación, el alcance se acota a un único MVP: un índice estructural de presión asistencial por patología × comunidad autónoma para 2023, complementado con un bloque descriptivo de tendencia nacional 2014-2023.

El proyecto sigue la metodología **CRISP-DM**, con especial énfasis en la honestidad metodológica: cada limitación de las fuentes se documenta como resultado, no se disimula, y la evaluación final se apoya en una validación cruzada por grupos (no en un K-Fold simple) precisamente para evitar sobreestimar lo que el modelo demuestra.

## Resultado principal

| | K-Fold simple | GroupKFold por patología | **GroupKFold por CCAA** |
|---|---|---|---|
| Mejora del modelo sobre el baseline | +8,6% | +5,4% (R² negativo) | **+0,7%** |

El índice separa **patologías** con solidez (dominadas por su perfil clínico nacional), pero la validación correcta -forzando al modelo a generalizar a comunidades autónomas nunca vistas- muestra que la contribución **territorial** es real pero mucho más modesta de lo que sugería una validación ingenua. Esta corrección, no la cifra optimista de partida, es el resultado central de la evaluación.

## Estructura del repositorio

```
.
├── README.md
└── docs/
    ├── assets/
    │   └── 05_mockup_frontal.png       # Mockup del frontal, usado para la presentación final
    ├── entregas/                       # Evolución del proyecto a lo largo del curso
    │   ├── 01_ideas_producto.md        # Entrega 1 - Ideas de producto iniciales
    │   ├── 02_datos_necesarios.md      # Entrega 2 - Idea seleccionada y datos necesarios
    │   ├── 03_modelo_datos.md          # Entrega 3 - Diseño del modelo de datos y capa gold
    │   ├── 04_analisis_modelado.md     # Entrega 4 - Diseño del análisis y estrategia de modelado
    │   └── 05_diseno_frontal.md        # Entrega 5 - Diseño del frontal y experiencia de usuario
    └── notebooks/                      # Notebooks de Google Colab
        ├── 1_TFM_Evolve
        ├── 1_TFM_Evolve_punto_2.5.ipynb
        └── 2_TFM_Evolve_punto_2.6.ipynb
        # (pendiente de subir el resto de notebooks)
```

> Las entregas 3, 4 y 5 recogen la evolución del proyecto durante el curso (incluidas las correcciones aplicadas tras el feedback recibido en cada fase) y ya están actualizadas para reflejar el diseño final: un único MVP construido íntegramente sobre datos reales, sin módulos que dependan de datos sintéticos.

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

## Demo

El mockup del frontal (`docs/assets/05_mockup_frontal.png`) corresponde a la presentación final del proyecto. El dashboard interactivo (HTML autocontenido, sin backend) que consume el índice ya calculado se añadirá al repositorio próximamente.

## Principios metodológicos

- **Sin datos sintéticos.** Todo el proyecto se construye y valida con fuentes 100% reales y públicas; los módulos que exigirían datos sintéticos (riesgo de reingreso, derivación multianual) se descartan y quedan documentados como diseño conceptual no implementado.
- **Limitación de fuente como resultado, no como nota al margen.** La imposibilidad de cruzar diagnóstico + territorio + tiempo en una sola fuente determina el alcance del proyecto desde la fase de comprensión de datos.
- **Validación honesta sobre validación favorable.** La evaluación final usa GroupKFold por CCAA precisamente porque es el esquema que puede desmentir el resultado más optimista, no el que mejor lo confirma.

## Limitaciones principales

- El índice no generaliza con solidez a comunidades autónomas nunca vistas (+0,7% de mejora bajo la validación correcta).
- El bloque de tendencia nacional es descriptivo: 10 puntos anuales no permiten forecasting con garantías estadísticas.
- No hay relación causal establecida entre las variables estructurales y la presión asistencial, solo asociación.
- No incluye reingreso a 30 días ni presión de derivación multianual: ninguna fuente pública ofrece esa granularidad sin recurrir a datos sintéticos.

## Público objetivo

Dirección médica de área sanitaria, jefaturas de servicio y gestión de camas del SNS, para apoyo a la **planificación estructural** — nunca como herramienta de decisión clínica individual sobre un paciente concreto.

## Pendiente de subir

- [ ] Memoria completa del TFM (PDF)
- [ ] Resto de notebooks (Colab)
- [ ] Dashboard interactivo (`.html`)
