# Workshop-1

Diseño de un Modelo Dimensional de Datos (Esquema en Estrella).

A continuación se presenta el modelo dimensional del proyecto:

```text
                        +--------------------+
                        |   dim_technology   |
                        +--------------------+
                        | technology_id (PK) |
                        | technology_name    |
                        +---------+----------+
                                  |
                                  |
+-------------------+             |             +--------------------+
|   dim_candidate   |             |             |   dim_seniority    |
+-------------------+             v             +--------------------+
| candidate_id (PK) |   +-------------------+   | seniority_id  (PK) |
| first_name        |-->| fact_applications |<--| seniority_level    |
| last_name         |   +-------------------+   +--------------------+
| email             |   | application_id(PK)|
                        | candidate_id  (FK)|
                        | technology_id (FK)|
                        | seniority_id  (FK)|
                        | location_id   (FK)|
                        | date_id       (FK)|
                        | code_challenge... |
                        | technical_inter...|
                        | is_hired          |
                        +---------+---------+
                                  ^
                                  |
            +---------------------+---------------------+
            |                                           |
+-----------+-------+                       +-----------+--------+
|   dim_location    |                       |      dim_date      |
+-------------------+                       +--------------------+
| location_id  (PK) |                       | date_id       (PK) |
| country           |                       | full_date          |
+-------------------+                       | year / month / day |
                                            +--------------------+
Arquitectura y Justificación del Diseño¿Por qué un modelo en estrella?Elegí un esquema en estrella porque simplifica enormemente los JOINs entre la tabla de hechos (fact_applications) y las dimensiones.
En lugar de anidar múltiples niveles de relaciones, cada dimensión se conecta directamente a la tabla central, lo que se traduce en consultas más rápidas y más fáciles de leer — algo clave tanto si el análisis se hace desde SQL como desde Pandas o una herramienta de BI.Definiendo el granoAntes de diseñar las tablas, definí qué representa cada fila de la fact table: una postulación individual de un candidato en una fecha específica.
Esta decisión es la base de todo el modelo, porque permite calcular con precisión promedios de puntaje, tasas de contratación y métricas de experiencia sin ambigüedad sobre qué se está midiendo.Cómo se organizaron las dimensionesdim_candidate: Separa la información personal (first_name, last_name, email) del resto del modelo.
Esto no solo evita duplicar datos demográficos, sino que también aísla la información de identificación personal (PII).dim_location y dim_technology: Normalizan los nombres de países y tecnologías para evitar la redundancia de texto en la tabla de hechos.dim_seniority: Categoriza los niveles de experiencia (Trainee, Junior, Mid-Level, Senior, Lead) basándose en los años de experiencia (YoE) para tratar valores faltantes o inconsistentes.dim_date: Descompone la fecha de aplicación en year, month y day para optimizar las consultas temporales y agregaciones por año.Proceso ETL (Extracción, Transformación y Carga)El pipeline ETL procesa el conjunto de datos crudos de candidatos, aplica transformaciones de calidad de datos y lógica de negocio, y carga los datos estructurados en un Data Warehouse local en SQLite.a. Extracción (Extract)Fuente: Archivo candidates.csv que contiene los registros crudos de las postulaciones.Implementación: El conjunto de datos se ingiere directamente en memoria utilizando Pandas (pd.read_csv()) para un procesamiento eficiente por lotes.b. Transformación (Transform)La etapa de transformación limpia registros no válidos, calcula campos de lógica de negocio y normaliza los datos en una estructura de Esquema en Estrella:Limpieza de Datos:Ajuste de Escala: Los valores de Code Challenge Score en escala de 0 a 100 se dividieron entre 10 para unificar todos los puntajes a una escala estándar de 0 a 10.Imputación de Seniority: Los registros de seniority faltantes se calcularon e imputaron automáticamente en función de los Años de Experiencia (YoE): Trainee (<1 año), Junior (<3 años), Mid-Level (<6 años), Senior (<10 años) y Lead (>=10 años).Filtro de Outliers: Se filtraron los valores irreales de experiencia (años negativos o valores no válidos como 99).Aplicación de la Regla de Contratación ("HIRED"):Un candidato se marca como contratado (is_hired = 1) si y solo si:$$\text{Code Challenge Score} \ge 7 \quad \text{Y} \quad \text{Technical Interview} \ge 7$$Los candidatos que no superan alguno de los dos umbrales se marcan como is_hired = 0.Modelado Dimensional y Mapeo:Se extraen los valores únicos para generar las tablas de dimensión normalizadas (dim_candidate, dim_location, dim_technology, dim_seniority, dim_date).Se generan claves subrogadas autoincrementables (_id) para cada dimensión y se mapean de regreso a la tabla de hechos central (fact_applications).c. Carga (Load)Destino: Base de datos relacional SQLite (dw_candidates.db).Ejecución: Los DataFrames transformados se persisten en la base de datos relacional usando to_sql() con if_exists='replace' para garantizar la idempotencia del pipeline.Integridad de Datos: Se ejecutan validaciones de recuento de filas tras la carga para confirmar cero pérdida de datos (por ejemplo, 50,000 registros de postulaciones verificados exitosamente en fact_applications).KPIs y VisualizacionesTodas las visualizaciones analíticas y de KPIs se generan directamente desde el Data Warehouse (dw_candidates.db) mediante consultas SQL relacionales conectadas con las tablas de dimensión.Consultas SQL Ejecutadas:Contrataciones por Tecnología (Gráfico de Pastel):SQLSELECT dt.technology_name, COUNT(fa.application_id) AS total_hires

FROM fact_applications fa
JOIN dim_technology dt ON fa.technology_id = dt.technology_id
WHERE fa.is_hired = 1
GROUP BY dt.technology_name
ORDER BY total_hires DESC;
Contrataciones por Año (Gráfico de Barras Horizontal):SQLSELECT dd.year, COUNT(fa.application_id) AS total_hires
FROM fact_applications fa
JOIN dim_date dd ON fa.date_id = dd.date_id
WHERE fa.is_hired = 1
GROUP BY dd.year;
Contrataciones por Seniority (Gráfico de Barras):SQLSELECT ds.seniority_level, COUNT(fa.application_id) AS total_hires
FROM fact_applications fa
JOIN dim_seniority ds ON fa.seniority_id = ds.seniority_id
WHERE fa.is_hired = 1
GROUP BY ds.seniority_level;
Contrataciones por País a lo largo del Tiempo (Gráfico Multilínea):SQLSELECT dd.year, dl.country, COUNT(fa.application_id) AS total_hires
FROM fact_applications fa
JOIN dim_location dl ON fa.location_id = dl.location_id
JOIN dim_date dd ON fa.date_id = dd.date_id
WHERE fa.is_hired = 1 AND dl.country IN ('USA', 'Brazil', 'Colombia', 'Ecuador')
GROUP BY dd.year, dl.country;
💡 Hallazgos Clave de Negocio (Insights)Tasa General de Contratación: De las 50,000 postulaciones, aproximadamente el 13.2% (6,599 candidatos) cumplieron con el umbral de contratación ($\ge 7$ tanto en la prueba técnica como en la entrevista).Tecnologías Top: Las tecnologías que generaron los mayores volúmenes de contratación fueron Development - CMS Backend y Sales, seguidas muy de cerca por React Frontend y Client Success.Demanda por Seniority: Tras la imputación de seniority por años de experiencia (YoE), las contrataciones se concentraron fuertemente en los niveles Lead, Senior y Mid-Level.Estabilidad Geográfica: Los mercados regionales clave (USA, Brasil, Colombia y Ecuador) mostraron un rendimiento de contratación constante entre 2021 y 2025.
