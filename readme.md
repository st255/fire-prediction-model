# Predicción de Incendios Forestales

Modelo de clasificación basado en Random Forest para evaluar el riesgo de incendios forestales en Castilla y León mediante la integración de registros históricos, datos meteorológicos dinámicos y mapas de vegetación.

## Descripción

El sistema combina tres fuentes principales de datos para estimar la probabilidad de incendio:
- Registros históricos de conatos e incendios (2006-2023).
- Series temporales meteorológicas para calcular el estrés hídrico y térmico del terreno.
- Datos de cobertura vegetal y grado de inflamabilidad de las especies presentes.

## Estado del Proyecto

### Tareas Completadas

- **Normalización espacial de registros de incendios**: Conversión de coordenadas UTM (ED50 y ETRS89 en husos 29, 30 y 31) al sistema global WGS84 (EPSG:4326).
- **Procesamiento de vegetación (MFE)**: Mapeo ordinal de riesgo por especies de flora, tipo de bosque, distribución y estado de desarrollo.
- **Ingeniería de características meteorológicas**: Extracción de variables desde archivos NetCDF mediante ventanas móviles (promedios y acumulados de lluvia a 7, 30, 60 y 90 días; temperatura, humedad, radiación y viento).
- **Generación de muestreo negativo**: Creación de puntos de control sin incendio (proporción 3:1) delimitados únicamente dentro de polígonos forestales.
- **Construcción del dataset**: Consolidación e integración espacio-temporal de datos meteorológicos y forestales en `dataset.csv`.
- **Configuración del modelo**: Definición inicial de `RandomForestClassifier` con balanceo de clases.

### Tareas Pendientes

- **Estrategia de división de datos**: Implementación de una separación temporal (por años) para evitar la fuga de información (*data leakage*).
- **Entrenamiento y optimización**: Ajuste de hiperparámetros mediante validación cruzada.
- **Evaluación y métricas**: Cálculo de ROC-AUC, F1-Score, matriz de confusión y análisis de importancia de variables (SHAP / Feature Importance).
- **Inferencia**: Módulo para la proyección de riesgo en tiempo real y generación de mapas de probabilidad.

## Datos de Entrada

| Fuente | Formato | Contenido |
| --- | --- | --- |
| Registros de Incendios | `.xlsx` | Ubicación y fecha de detección de incendios. |
| Mapa Forestal de España | `.shp` | Tipo de masa forestal, especies y cobertura. |
| E-OBS Reanalysis | `.nc` | Datos diarios de temperatura, precipitación, humedad, radiación y viento. |

## Requisitos

- Python 3.12+
- `pandas`
- `geopandas`
- `xarray`
- `scikit-learn`
- `pyproj`
- `netCDF4`