# Proyecto MEDORO 7 – SQL Server 2022 + Power BI

Este proyecto optimiza y automatiza el análisis de tiempos de producción, preparación, parada y mantenimiento en planta, basado en registros del sistema SQL Server. El objetivo es entregar una vista limpia y final que pueda ser utilizada directamente por el equipo de planta y visualizada en Power BI, sin necesidad de procesamiento manual adicional.

## 🔍 Estructura del proyecto

### ✅ `vista_ConCubo_Medoro7_Limpia`

**Base estructurada y corregida sobre la que se construye todo el análisis.**

* Parte de la tabla original `ConCubo`
* Corrige el desfase histórico en las fechas (resta 2 días)
* Convierte fechas a texto para evitar jerarquías automáticas en Power BI
* Extrae `ID_Limpio` numérico desde la columna `ID`
* Filtra por `Renglon = 201` y solo datos del año 2025
* Calcula duración total en horas y la separa según tipo de estado
* Convierte `CantidadBuenosProducida` a tipo numérico

### ✅ `vista_MedoroResumen7_2025`

**Vista resumen para visualización y análisis en Power BI**

* Reutiliza campos corregidos desde la vista anterior
* Incluye fecha legible y columnas de duración discriminadas
* Añade columna `Nro` con `ROW_NUMBER()` para ordenar eventos cronológicamente
* Incluye cantidad de buenos producidos

### ✅ `vista_MedoroSecuencias7_2025`

**Incorpora la lógica de secuencia y flag por bloque**

* Calcula cambios de OT y reinicios por interrupciones
* Usa `LAG()` y `SUM(Flag)` para detectar cortes de continuidad
* Agrega columnas: `Secuencia`, `FlagSecuencia`

---

## 🏭 Instalación en Planta

### ✅ Vista final recomendada para Power BI: `vista_MedoroResumen7_Final_2025`

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

    -- Cálculos de horas
    DATEDIFF(SECOND, m.Inicio_Corregido, m.Fin_Corregido) / 3600.0 AS Total_Horas,
    CASE WHEN m.Estado = 'Producción' THEN DATEDIFF(SECOND, m.Inicio_Corregido, m.Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Produccion,
    CASE WHEN m.Estado = 'Preparación' THEN DATEDIFF(SECOND, m.Inicio_Corregido, m.Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Preparacion,
    CASE WHEN m.Estado = 'Maquina Parada' THEN DATEDIFF(SECOND, m.Inicio_Corregido, m.Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Parada,
    CASE WHEN m.Estado = 'Mantenimiento' THEN DATEDIFF(SECOND, m.Inicio_Corregido, m.Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Mantenimiento,

    m.CantidadBuenosProducida,

    -- Secuencia cronológica por OT
    ROW_NUMBER() OVER (PARTITION BY m.ID_Limpio ORDER BY m.Inicio_Corregido ASC) AS Nro,

    -- Sacabocado
    vu.saccod1

FROM vista_ConCubo_Medoro7_Limpia AS m
LEFT JOIN TablaVinculadaUNION AS vu
    ON TRY_CAST(vu.OP AS INT) = m.ID_Limpio
WHERE 
    m.Renglon = 201 AND
    YEAR(m.Inicio_Corregido) = 2025;
```

Esta vista es la que se debe usar como fuente de datos principal en Power BI.

---

### 📊 Medidas DAX recomendadas en Power BI:

```dax
Horas_Preparacion_Total =
    SUM(vista_MedoroResumen7_Final_2025[Horas_Preparacion])

Horas_Produccion_Total =
    SUM(vista_MedoroResumen7_Final_2025[Horas_Produccion])

Horas_Parada_Total =
    SUM(vista_MedoroResumen7_Final_2025[Horas_Parada])

Horas_Mantenimiento_Total =
    SUM(vista_MedoroResumen7_Final_2025[Horas_Mantenimiento])

Cantidad_Producida_Total =
    SUM(vista_MedoroResumen7_Final_2025[CantidadBuenosProducida])
```

Estas medidas se colocan en visuales tipo tarjeta para tener los indicadores clave siempre visibles.

---

**Marcelo Fabián López – Proyecto Medoro 7**
