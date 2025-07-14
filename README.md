# Proyecto Medoro 7 – Optimización de tiempos y producción (2025)

## 📌 Objetivo

Automatizar el análisis de tiempos de preparación, producción, parada y mantenimiento para órdenes de trabajo en la planta, corrigiendo errores históricos y validando métricas clave. El proyecto permite que Power BI se conecte directamente a vistas SQL limpias, confiables y validadas con la supervisión de planta.

---

## 🧱 Vista Base: `vista_ConCubo_Medoro7_Limpia`

**Fuente:** `ConCubo`

**Qué hace:**

* Corrige fechas (`Inicio`, `Fin`) restando 2 días.
* Convierte fechas a texto para evitar jerarquías automáticas en Power BI.
* Extrae `ID_Limpio` como entero desde el `ID` original.
* Filtra por `Renglon = 201` y año 2025.
* Calcula duración total en horas.
* Separa horas por tipo de estado.
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

## 📊 Vista Resumen con Secuencia: `vista_MedoroResumen7_2025`

**Fuente:** `vista_ConCubo_Medoro7_Limpia`

**Qué hace:**

* Agrega columna `Nro` con `ROW_NUMBER()` para secuenciar los eventos por OT.

**Código:**

```sql
CREATE OR ALTER VIEW vista_MedoroResumen7_2025 AS
SELECT
    ID,
    ID_Limpio,
    Renglon,
    Estado,
    Inicio_Corregido,
    Fin_Corregido,
    Inicio_Legible_Texto,
    Fin_Legible_Texto,
    CONVERT(DATE, Inicio_Corregido) AS Fecha,
    DATEDIFF(SECOND, Inicio_Corregido, Fin_Corregido) / 3600.0 AS Total_Horas,
    CASE WHEN Estado = 'Producción' THEN DATEDIFF(SECOND, Inicio_Corregido, Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Produccion,
    CASE WHEN Estado = 'Preparación' THEN DATEDIFF(SECOND, Inicio_Corregido, Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Preparacion,
    CASE WHEN Estado = 'Maquina Parada' THEN DATEDIFF(SECOND, Inicio_Corregido, Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Parada,
    CASE WHEN Estado = 'Mantenimiento' THEN DATEDIFF(SECOND, Inicio_Corregido, Fin_Corregido) / 3600.0 ELSE 0 END AS Horas_Mantenimiento,
    CantidadBuenosProducida,
    ROW_NUMBER() OVER (PARTITION BY ID_Limpio ORDER BY Inicio_Corregido ASC) AS Nro
FROM vista_ConCubo_Medoro7_Limpia
WHERE Renglon = 201 AND YEAR(Inicio_Corregido) = 2025;
```

---

## 🧩 Vista Extendida con Sacabocado: `vista_MedoroResumen7_2025_ConSacco`

**Unión:** `ID_Limpio` ↔ `TRY_CAST(OP AS INT)`

**Código:**

```sql
CREATE OR ALTER VIEW vista_MedoroResumen7_2025_ConSacco AS
SELECT 
    R.*,
    VU.saccod1
FROM vista_MedoroResumen7_2025 R
LEFT JOIN TablaVinculadaUNION VU
    ON TRY_CAST(VU.OP AS INT) = R.ID_Limpio;
```

---

## 📐 Medidas DAX en Power BI (estilo Medoro 5–6)

```DAX
Horas_Preparacion_Total = SUM(vista_MedoroResumen7_2025_ConSacco[Horas_Preparacion])
Horas_Produccion_Total = SUM(vista_MedoroResumen7_2025_ConSacco[Horas_Produccion])
Horas_Parada_Total = SUM(vista_MedoroResumen7_2025_ConSacco[Horas_Parada])
Horas_Mantenimiento_Total = SUM(vista_MedoroResumen7_2025_ConSacco[Horas_Mantenimiento])
Cantidad_Producida_Total = SUM(vista_MedoroResumen7_2025_ConSacco[CantidadBuenosProducida])
```

---

## ✅ Resultado

* Todas las vistas están validadas con José.
* Los cálculos coinciden con el control manual.
* Las secuencias por OT ya están generadas.
* El campo `saccod1` ya está incluido.

📊 **Ya está listo para conectar a Power BI y crear visuales automáticos.**
