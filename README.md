# Proyecto MEDORO 7 – Optimización de Tiempos de Producción y Preparación (2025)

Este proyecto mejora drásticamente el análisis de eficiencia de producción en planta. A partir de datos sin estructura y registros duplicados, se logra generar vistas limpias, confiables y conectables directamente a Power BI.

---

## 🔄 Origen del problema

* Las tablas `ConCubo` y `TablaVinculadaUNION` contenían datos con fechas mal registradas, IDs mezclados (texto y número), y sin posibilidad de distinguir secuencias de eventos dentro de una misma orden de trabajo (OT).
* La planta no podía analizar correctamente los tiempos de preparación, producción, parada y mantenimiento.
* Todo el análisis se hacía de forma manual en Excel.

---

## 📁 Vistas creadas

### 1. `vista_ConCubo_Medoro7_Limpia`

Esta es la base estructurada y corregida sobre la que se construye todo el análisis.

**Qué hace:**

* Usa `ConCubo` como origen.
* Corrige fechas restando 2 días.
* Convierte fechas a texto legible (evita jerarquías automáticas en Power BI).
* Extrae `ID_Limpio` como versión numérica del `ID`.
* Filtra por `Renglon = 201` y solo datos del año 2025.
* Calcula duración total en horas y la separa por tipo de estado.
* Convierte `CantidadBuenosProducida` a tipo numérico.

**Código:**

```sql
CREATE OR ALTER VIEW vista_ConCubo_Medoro7_Limpia AS
SELECT
    ID,
    TRY_CAST(SUBSTRING(ID, PATINDEX('%[0-9]%', ID), LEN(ID)) AS INT) AS ID_Limpio,
    Renglon,
    Estado,
    DATEADD(DAY, -2, TRY_CAST(Inicio AS DATETIME)) AS Inicio_Corregido,
    DATEADD(DAY, -2, TRY_CAST(Fin AS DATETIME)) AS Fin_Corregido,
    CONVERT(VARCHAR(16), DATEADD(DAY, -2, TRY_CAST(Inicio AS DATETIME)), 120) AS Inicio_Legible_Texto,
    CONVERT(VARCHAR(16), DATEADD(DAY, -2, TRY_CAST(Fin AS DATETIME)), 120) AS Fin_Legible_Texto,
    CONVERT(DATE, DATEADD(DAY, -2, TRY_CAST(Inicio AS DATETIME))) AS Fecha,
    DATEDIFF(SECOND, TRY_CAST(Inicio AS DATETIME), TRY_CAST(Fin AS DATETIME)) / 3600.0 AS Total_Horas,
    CASE WHEN Estado = 'Producción' THEN DATEDIFF(SECOND, TRY_CAST(Inicio AS DATETIME), TRY_CAST(Fin AS DATETIME)) / 3600.0 ELSE 0 END AS Horas_Produccion,
    CASE WHEN Estado = 'Preparación' THEN DATEDIFF(SECOND, TRY_CAST(Inicio AS DATETIME), TRY_CAST(Fin AS DATETIME)) / 3600.0 ELSE 0 END AS Horas_Preparacion,
    CASE WHEN Estado = 'Parada' THEN DATEDIFF(SECOND, TRY_CAST(Inicio AS DATETIME), TRY_CAST(Fin AS DATETIME)) / 3600.0 ELSE 0 END AS Horas_Parada,
    CASE WHEN Estado = 'Mantenimiento' THEN DATEDIFF(SECOND, TRY_CAST(Inicio AS DATETIME), TRY_CAST(Fin AS DATETIME)) / 3600.0 ELSE 0 END AS Horas_Mantenimiento,
    TRY_CAST(CantidadBuenosProducida AS FLOAT) AS CantidadBuenosProducida
FROM ConCubo
WHERE
    Renglon = 201
    AND TRY_CAST(Inicio AS DATETIME) >= '2025-01-01'
    AND TRY_CAST(Inicio AS DATETIME) < '2026-01-01'
    AND ISNUMERIC(SUBSTRING(ID, PATINDEX('%[0-9]%', ID), LEN(ID))) = 1;
```

---

### 2. `vista_MedoroResumen7_Final_2025`

Esta es la vista final que integra todo:

* Correcciones de fecha
* Cálculos por tipo de estado
* Cantidades producidas
* Secuencia ordenada
* Sacabocado (`saccod1`) desde `TablaVinculadaUNION`

**Código:**

```sql
CREATE OR ALTER VIEW vista_MedoroResumen7_Final_2025 AS
SELECT
    m.ID,
    m.ID_Limpio,
    m.Renglon,
    m.Estado,
    m.Inicio_Corregido,
    m.Fin_Corregido,
    m.Inicio_Legible_Texto,
    m.Fin_Legible_Texto,
    CONVERT(DATE, m.Inicio_Corregido) AS Fecha,
    DATEDIFF(SECOND, m.Inicio_Corregido, m.Fin_Corregido) / 3600.0 AS Total_Horas,
    CASE WHEN m.Estado = 'Producción' THEN DATEDIFF(SECOND, m.Inicio_Corregido, m.Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Produccion,
    CASE WHEN m.Estado = 'Preparación' THEN DATEDIFF(SECOND, m.Inicio_Corregido, m.Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Preparacion,
    CASE WHEN m.Estado = 'Maquina Parada' THEN DATEDIFF(SECOND, m.Inicio_Corregido, m.Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Parada,
    CASE WHEN m.Estado = 'Mantenimiento' THEN DATEDIFF(SECOND, m.Inicio_Corregido, m.Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Mantenimiento,
    m.CantidadBuenosProducida,
    ROW_NUMBER() OVER (PARTITION BY m.ID_Limpio ORDER BY m.Inicio_Corregido ASC) AS Nro,
    v.saccod1
FROM vista_ConCubo_Medoro7_Limpia AS m
LEFT JOIN TablaVinculadaUNION AS v ON TRY_CAST(v.OP AS INT) = m.ID_Limpio
WHERE
    m.Renglon = 201 AND
    YEAR(m.Inicio_Corregido) = 2025;
```

---

## 📊 Medidas DAX sugeridas para Power BI

```DAX
Horas_Produccion_Total = SUM(vista_MedoroResumen7_Final_2025[Horas_Produccion])
Horas_Preparacion_Total = SUM(vista_MedoroResumen7_Final_2025[Horas_Preparacion])
Horas_Parada_Total = SUM(vista_MedoroResumen7_Final_2025[Horas_Parada])
Horas_Mantenimiento_Total = SUM(vista_MedoroResumen7_Final_2025[Horas_Mantenimiento])
Cantidad_Producida_Total = SUM(vista_MedoroResumen7_Final_2025[CantidadBuenosProducida])
```

---

## 📈 Visuales sugeridos

* Tarjetas KPI con medidas anteriores
* Gráfico de barras: `ID_Limpio` vs Horas por tipo
* Gráfico de líneas: eficiencia diaria (por ejemplo, porcentaje de preparación)
* Tabla detallada: secuencia por evento con tiempo y cantidad

---

## 🏢️ Instalación en planta

1. Crear en SQL Server las dos vistas:

   * `vista_ConCubo_Medoro7_Limpia`
   * `vista_MedoroResumen7_Final_2025`

2. En Power BI:

   * Conectar directo a `vista_MedoroResumen7_Final_2025`
   * Crear medidas y visuales como se detalló

3. Comunicar al equipo:

> "Usen la vista `vista_MedoroResumen7_Final_2025` como origen. Contiene los datos corregidos, validados, con tiempos reales, secuencia y sacabocado. Ya no hace falta usar Excel ni manipular manualmente las tablas originales."

---

## 🚀 Impacto

* Eliminación de errores manuales
* Reducción de tiempos de análisis
* Validación completa con datos reales (OT 14620, 14626)
* Reutilizable por toda la empresa sin tocar datos originales

---

Cualquier modificación o mejora posterior (por ejemplo, incorporar otras máquinas) puede partir directamente desde esta arquitectura limpia.

---

📅 Proyecto creado por Marcelo F. López - 2025
