# Workshop 001: ETL Pipeline y Data Warehouse de Candidatos

Este proyecto es una solución de Data Engineering para procesar, limpiar y modelar los datos de 50,000 postulaciones de candidatos. El objetivo principal fue construir un pipeline ETL que tome los datos crudos, corrija errores de calidad de datos y los cargue en un modelo dimensional (Star Schema) en SQLite para consultar KPIs de contratación.

---

##  Arquitectura del Data Warehouse

**Esquema en Estrella (Star Schema)** centralizado en SQLite. La estructura permite hacer consultas rápidas para BI sin necesidad de unir demasiadas tablas.

```text
  dim_candidate (candidate_id, first_name, last_name, email)
        │
  dim_technology (technology_id, technology_name)
        │
  fact_applications (application_id, candidate_id, technology_id, 
                     seniority_id, location_id, date_id, 
                     code_challenge_score, technical_interview_score, is_hired)
        │
  dim_seniority (seniority_id, seniority_level)
        │
  dim_location (location_id, country)
        │
  dim_date (date_id, full_date, year, month, day)

Decisiones clave del modelo:
Grano: Cada fila de fact_applications representa una postulación individual de un candidato en una fecha determinada.

Privacidad (PII): Aislé los datos personales (first_name, last_name, email) en dim_candidate para no mezclar datos sensibles con las métricas del proceso.
Proceso ETL y Calidad de Datos
Durante la fase de transformación en Python (Pandas), apliqué estas reglas de limpieza antes de cargar la base de datos:

Ajuste de Escala: Identifiqué que algunos puntajes de Code Challenge Score venían en escala de 100. Los normalicé dividiendo entre 10 para llevar todo a una escala uniforme de 0 a 10.

Imputación de Seniority: Había registros sin nivel de seniority. Creé una regla que asigna la categoría según los años de experiencia (Yoe), por ejemplo: < 1 año como Trainee, < 3 como Junior, < 6 como Mid-Level, etc.

Filtro de Outliers: Se limpiaron valores irreales en la experiencia laboral (años negativos o valores desproporcionados como 99).

Criterio de Contratación (is_hired): Un candidato solo se marca como contratado (is_hired = 1) si obtuvo una nota mayor o igual a 7 tanto en la prueba técnica como en la entrevista.
Dashboard de KPIs
Las visualizaciones fueron generadas consultando la base de datos dw_candidates.db mediante SQLAlchemy y renderizadas en Matplotlib con un estilo visual personalizado.

Ejemplo de Consulta SQL Utilizada:
-- Contrataciones por tecnología
SELECT t.technology_name, COUNT(*) as contrataciones
FROM fact_applications f
JOIN dim_technology t ON f.technology_id = t.technology_id
WHERE f.is_hired = 1
GROUP BY t.technology_name 
ORDER BY contrataciones DESC;

  Hallazgos Clave de Negocio (Insights)

* **Efectividad del Proceso:** De las 50,000 postulaciones analizadas, aproximadamente el **13.2% (6,599 candidatos)** cumplieron simultáneamente con el criterio de corte ($\ge 7$ en prueba técnica y entrevista).
* **Demanda Tecnológica:** Las tecnologías con mayor volumen de contratación fueron **Development - CMS Backend** y **Sales**, seguidas muy de cerca por **React Frontend** y **Client Success**.
* **Distribución por Seniority:** Tras la imputación de datos basada en experiencia (`Yoe`), los niveles **Lead**, **Senior** y **Mid-Level** concentraron la mayor cantidad de contrataciones.
* **Estabilidad Geográfica:** Los mercados de USA, Brasil, Colombia y Ecuador mostraron una tendencia de contratación uniforme entre 2021 y 2025, con el pico esperado de corte en el volumen del año en curso (2026).



