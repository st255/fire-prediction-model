# Predicción de Incendios Forestales en Castilla y León

Modelo de clasificación que estima la probabilidad de que se produzca un incendio forestal en un punto y un día concretos de Castilla y León. Combina el registro histórico de incendios (2006-2023) con datos meteorológicos diarios, mapas de vegetación, relieve y población.

| | ROC-AUC temporal | ROC-AUC espacial |
| --- | :---: | :---: |
| **Mejor modelo (LightGBM ajustado)** | **0.803** | **0.767** |

- **Temporal**: capacidad de predecir incendios de años futuros que el modelo no ha visto.
- **Espacial**: capacidad de predecir incendios en zonas del mapa que el modelo no ha visto.

---

## Índice

1. [Fuentes de datos](#fuentes-de-datos)
2. [Construcción del dataset](#construcción-del-dataset)
3. [Estrategia de validación](#estrategia-de-validación)
4. [Comparativa de modelos](#comparativa-de-modelos)
5. [Ajuste del mejor modelo](#ajuste-del-mejor-modelo)
6. [Conclusiones](#conclusiones)
7. [Estructura del repositorio](#estructura-del-repositorio)
8. [Requisitos](#requisitos)

---

## Fuentes de datos

| Fuente | Origen | Formato | Contenido |
| --- | --- | --- | --- |
| Registro de incendios | [Ministerio de Agricultura, Pesca y Alimentación (MAPA)](https://servicio.mapa.gob.es/incendios/Search/Publico) | `.xlsx` | Fecha de detección y coordenadas UTM de cada incendio y conato (2006-2023). |
| Meteorología | [E-OBS v33.0e, Copernicus Climate Data Store](https://cds.climate.copernicus.eu/datasets/insitu-gridded-observations-europe?tab=overview) | `.nc` | Rejilla diaria de 0.1° con temperatura media, mínima y máxima, precipitación, humedad relativa, radiación solar, velocidad del viento y presión a nivel del mar. |
| Relieve | [E-OBS v33.0e, Copernicus Climate Data Store](https://cds.climate.copernicus.eu/datasets/insitu-gridded-observations-europe?tab=overview) | `.nc` | Elevación del terreno en la misma rejilla de 0.1°. |
| Vegetación | [Mapa Forestal de España (MITECO)](https://www.miteco.gob.es/es/cartografia-y-sig/ide/descargas/biodiversidad/mfe.html) | `.shp` | Polígonos forestales con especies, ocupación, estado de desarrollo, coberturas, tipo de bosque y modelo de combustible. |
| Población | [Padrón municipal (INE)](https://www.ine.es/dynt3/inebase/es/index.htm?padre=525) | `.csv` | Población anual de cada municipio de las nueve provincias. |
| Límites municipales | [Centro de Descargas del CNIG (IGN)](https://centrodedescargas.cnig.es) | `.shp` | Recintos municipales, usados para asignar cada punto a su municipio. |

---

## Construcción del dataset

Todo el proceso está en [`preprocessing.ipynb`](preprocessing.ipynb) y genera `dataset.csv`.

1. **Limpieza de incendios**: los registros usan dos sistemas de coordenadas (ED50 en los antiguos y ETRS89 en los modernos) en los husos 29, 30 y 31. Todos se convierten a WGS84 (EPSG:4326). Se descartan los registros sin coordenadas o con errores evidentes, y los que caen fuera de la región.
2. **Variables meteorológicas**: además del valor del día, se calculan ventanas móviles que resumen el estado del terreno en los días previos.
   - Precipitación acumulada a 7, 30 y 90 días.
   - Humedad media a 7 y 30 días.
   - Máximos y mínimos de temperatura y de viento en distintas ventanas.
   - Radiación acumulada a 7 y 30 días.
   - Índices derivados: viento/humedad, índice de propagación (viento × sequedad × temperatura), déficit de humedad a 30 días y días secos consecutivos.
   - Codificación cíclica del día del año (seno y coseno).
3. **Relieve**: a partir de la elevación se obtienen la pendiente y la orientación sur del terreno.
4. **Vegetación**: las especies se transforman en un índice de riesgo de combustible (de 0 a 5), ponderado por la ocupación de cada especie en el polígono. El estado de desarrollo y la distribución de la masa se codifican de forma ordinal, y el tipo de bosque con one-hot.
5. **Población**: cada punto se asigna a su municipio y se le une la población del padrón de ese año.
6. **Muestreo de negativos**: por cada incendio se generan dos puntos sin incendio, siempre dentro de zonas forestales:
   - uno con **la misma fecha** que el incendio, para que el modelo aprenda a distinguir lugares;
   - otro con **una fecha aleatoria**, para que aprenda a distinguir días.

El resultado tiene **82 537 registros** (27 468 incendios, el 33 %) y **48 variables**. El número de incendios varía bastante entre años, con picos en 2009, 2011, 2012 y 2017 (análisis en [`dataset_analysis.ipynb`](dataset_analysis.ipynb)).

![Registros por año](images/events_per_year.png)

---

## Estrategia de validación

Una división aleatoria de los datos daría resultados demasiado optimistas: días consecutivos y puntos cercanos comparten casi la misma meteorología. Por eso se usan dos pruebas independientes.

- **Validación temporal**: para cada año entre 2015 y 2023, el modelo se entrena solo con datos anteriores y se evalúa en ese año. Entre el entrenamiento y el test se dejan **90 días de margen**, la ventana móvil más larga, para que ninguna variable del test se calcule con días del entrenamiento.
- **Validación espacial**: el mapa se divide en bloques de 0.3° (unos 30 km) que se reparten en **5 grupos**. El modelo se evalúa en cada grupo tras entrenar con el resto, así que siempre predice zonas que no ha visto.

La métrica principal es el **ROC-AUC**: 0.5 equivale a predecir al azar y 1 a un modelo perfecto.

---

## Comparativa de modelos

En [`models.ipynb`](models.ipynb) se evalúan 12 modelos de 8 familias distintas con las dos pruebas.

| Familia | Modelos |
| --- | --- |
| Referencia | Dummy (predice siempre la proporción de incendios) |
| Lineal | Regresión logística |
| Probabilística | Naive Bayes |
| Basada en instancias | K-Nearest Neighbors |
| Árbol simple | Decision Tree |
| Bagging | Random Forest, Extra Trees |
| Boosting | HistGradientBoosting, XGBoost, LightGBM, CatBoost |
| Red neuronal | Multilayer Perceptron |

### Resultados

| Modelo | ROC-AUC temporal | ROC-AUC espacial | PR-AUC temporal | PR-AUC espacial | Tiempo por entrenamiento |
| --- | :---: | :---: | :---: | :---: | :---: |
| **LightGBM** | **0.797** | **0.767** | **0.651** | **0.616** | 5 s |
| XGBoost | 0.796 | 0.764 | 0.650 | 0.615 | 24 s |
| CatBoost | 0.795 | 0.766 | 0.648 | 0.615 | 31 s |
| HistGradientBoosting | 0.791 | 0.761 | 0.641 | 0.609 | 10 s |
| Random Forest | 0.773 | 0.765 | 0.616 | 0.612 | 21 s |
| Extra Trees | 0.764 | 0.760 | 0.601 | 0.605 | 9 s |
| MLP | 0.764 | 0.757 | 0.599 | 0.602 | 12 s |
| KNN | 0.735 | 0.730 | 0.562 | 0.564 | < 1 s |
| Decision Tree | 0.721 | 0.696 | 0.539 | 0.515 | 2 s |
| Regresión logística | 0.715 | 0.711 | 0.526 | 0.536 | 2 s |
| Naive Bayes | 0.659 | 0.653 | 0.450 | 0.460 | < 1 s |
| Dummy | 0.500 | 0.500 | 0.315 | 0.332 | - |

Los modelos de **boosting** son los mejores en las dos pruebas. Las diferencias entre ellos son muy pequeñas, y por debajo hay escalones claros entre familias.

![ROC-AUC por modelo](images/models_auc.png)

El orden de las familias se mantiene en todos los años y en todas las zonas. Hay años más difíciles para todos los modelos (2017 y 2022), lo que apunta a una dificultad de los datos y no de un modelo concreto.

![ROC-AUC por año y por zona](images/models_auc_per_year_fold.png)

Curvas ROC del mejor modelo de cada familia:

![Curvas ROC por familia](images/models_roc_families.png)

Los mejores modelos rinden algo peor en zonas nuevas que en años nuevos: quedan por debajo de la diagonal.

![ROC-AUC temporal frente a espacial](images/models_temporal_vs_spatial.png)

LightGBM es el mejor y además uno de los más rápidos:

![Rendimiento frente a tiempo de entrenamiento](images/models_training_time.png)

---

## Ajuste del mejor modelo

En [`bestmodel.ipynb`](bestmodel.ipynb) se ajustan los hiperparámetros de LightGBM con **Optuna** (60 combinaciones). Para obtener una puntuación honesta:

- el ajuste solo usa **datos anteriores a 2015**, con su propia validación temporal (2012-2014) y espacial;
- la evaluación final se hace con **datos de 2015 a 2023**, que el ajuste nunca ve.

![Historial del ajuste](images/tuning_history.png)

| Hiperparámetro | Original | Ajustado |
| --- | :---: | :---: |
| Número de árboles | 1000 | 1700 |
| Tasa de aprendizaje | 0.02 | 0.005 |
| Hojas por árbol | 63 | 213 |
| Muestras mínimas por hoja | 20 | 17 |
| Filas por árbol | 80 % | 98 % |
| Variables por árbol | 70 % | 30 % |

El umbral de decisión también se elige con los datos de ajuste: el que maximiza F1 es **0.369**, en lugar del 0.5 por defecto.

| | ROC-AUC temporal | ROC-AUC espacial | PR-AUC temporal | PR-AUC espacial | F1 temporal | F1 espacial |
| --- | :---: | :---: | :---: | :---: | :---: | :---: |
| LightGBM original | 0.797 | 0.763 | 0.651 | 0.603 | 0.612 | 0.575 |
| **LightGBM ajustado** | **0.803** | **0.767** | **0.659** | **0.613** | **0.613** | **0.590** |

> En la prueba espacial solo se evalúan datos de 2015 en adelante. Por eso el modelo original saca 0.763 y no el 0.767 de la comparativa.

El modelo ajustado mejora al original en los 9 años y en las 5 zonas:

![Original frente a ajustado por año y por zona](images/bestmodel_per_year_fold.png)

Variables más importantes del modelo final:

![Importancia de variables](images/feature_importance.png)

El modelo final, entrenado con todos los datos, se guarda en `best_model.txt` y se carga con `lightgbm.Booster(model_file="best_model.txt")`.

---

## Conclusiones

- **El riesgo de incendio se puede predecir con una capacidad moderada-alta**: el modelo final alcanza un ROC-AUC de 0.80 en años futuros y de 0.77 en zonas nuevas.
- **El boosting es la mejor familia**: LightGBM, XGBoost, CatBoost e HistGradientBoosting quedan prácticamente empatados, con diferencias menores que la variación entre años. LightGBM es la mejor opción porque entrena entre 5 y 6 veces más rápido que XGBoost o CatBoost.
- **Las relaciones entre variables son importantes**: la regresión logística queda unos 0.08 puntos de AUC por debajo del boosting, así que el riesgo no se explica bien sumando el efecto de cada variable por separado.
- **Generalizar a zonas nuevas es más difícil que a años nuevos**: los mejores modelos pierden unos 0.03 puntos de AUC en la prueba espacial.
- **El ajuste de hiperparámetros aporta poco**: +0.005 de AUC aproximadamente. Es una mejora constante, pero indica que el modelo está cerca de su límite con las variables actuales. Las próximas mejoras deberían venir de los datos y no del modelo.
- **El relieve y la población pesan más que la meteorología**: pendiente, elevación, población y orientación son las variables más importantes. Puede deberse en parte a cómo se generan los negativos (puntos aleatorios dentro de zonas forestales), así que conviene comprobar que el modelo no está aprendiendo simplemente dónde suelen registrarse incendios.

### Próximos pasos

- Revisar el muestreo de negativos para reducir el posible sesgo espacial.
- Añadir variables nuevas, como distancia a carreteras y núcleos urbanos, usos del suelo o índices de vegetación por satélite.
- Crear un módulo de inferencia que genere mapas diarios de probabilidad de incendio.

---

## Estructura del repositorio

```
├── data/                     # Datos originales (no incluidos)
├── images/                   # Gráficos usados en este documento
├── preprocessing.ipynb       # Limpieza, cruce de fuentes y generación de dataset.csv
├── dataset_analysis.ipynb    # Análisis exploratorio del dataset
├── models.ipynb              # Comparativa de modelos con validación temporal y espacial
├── bestmodel.ipynb           # Ajuste de LightGBM y evaluación final
├── dataset.csv               # Dataset final
└── best_model.txt            # Modelo final entrenado
```

Los notebooks se ejecutan en este orden: `preprocessing` → `dataset_analysis` → `models` → `bestmodel`.

---

## Requisitos

- Python 3.12
- Datos: `pandas`, `numpy`, `geopandas`, `xarray`, `netCDF4`, `pyproj`
- Modelos: `scikit-learn`, `xgboost`, `lightgbm`, `catboost`, `optuna`
- Gráficos: `matplotlib`, `seaborn`
