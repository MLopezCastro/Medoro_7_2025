## 📃 Proyecto: Medoro 7 – Análisis de Tiempos con Validación Real (2025)

### 🌍 Contexto General

Este proyecto nace para resolver un problema crítico en la planta: los tiempos de preparación, producción, parada y mantenimiento estaban mal calculados, y los datos originales venían de tablas sucias, con desfases de fechas, IDs no normalizados y errores de estructura. Todo se validó contra registros manuales provistos por José.

---

### 🔧 Objetivo

Crear un modelo SQL robusto y reutilizable que:

* Corrija el desfase de fechas.
* Normalice los IDs (ID\_Limpio).
* Separe los tiempos por tipo (producción, preparación, parada, mantenimiento).
* Permita ver secuencias de eventos dentro de una misma OT.
* Agregue el número de sacabocado (saccod1) desde la tabla `TablaVinculadaUNION`.
* Sea 100% compatible con Power BI.

---

## 🔖 Estructura de Vistas SQL

### 1. `vista_ConCubo_Medoro7_Limpia`

Vista base que:

* Parte de la tabla `ConCubo`.
* Corrige fechas con `DATEADD(DAY, -2, ...)`.
* Extrae `ID_Limpio` como versión numérica segura del campo `ID`.
* Filtra por `Renglon = 201` y año 2025.
* Convierte fechas a formato legible.
* Calcula horas por tipo de estado.
* Convierte `CantidadBuenosProducida` a `FLOAT`.

**Código**:

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
    Renglon = 201 AND
    TRY_CAST(Inicio AS DATETIME) >= '2025-01-01' AND
    TRY_CAST(Inicio AS DATETIME) < '2026-01-01' AND
    ISNUMERIC(SUBSTRING(ID, PATINDEX('%[0-9]%', ID), LEN(ID))) = 1;
```

---

### 2. `vista_MedoroResumen7_Final_2025`

Vista final y lista para Power BI. Contiene:

* Todo lo de la vista limpia anterior.
* Secuencia (`ROW_NUMBER()` por ID).
* El `saccod1` traído desde `TablaVinculadaUNION`, validado con `ISNUMERIC` para evitar errores.

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
    VU.saccod1
FROM vista_ConCubo_Medoro7_Limpia AS m
LEFT JOIN TablaVinculadaUNION AS VU
    ON ISNUMERIC(VU.OP) = 1 AND TRY_CAST(VU.OP AS INT) = m.ID_Limpio
WHERE
    m.Renglon = 201 AND YEAR(m.Inicio_Corregido) = 2025;
```

---

## 📊 DAX para Power BI (medidas)

```DAX
Horas_Produccion_Total = SUM(vista_MedoroResumen7_Final_2025[Horas_Produccion])
Horas_Preparacion_Total = SUM(vista_MedoroResumen7_Final_2025[Horas_Preparacion])
Horas_Parada_Total = SUM(vista_MedoroResumen7_Final_2025[Horas_Parada])
Horas_Mantenimiento_Total = SUM(vista_MedoroResumen7_Final_2025[Horas_Mantenimiento])
Cantidad_Producida_Total = SUM(vista_MedoroResumen7_Final_2025[CantidadBuenosProducida])
%Tiempo_Preparacion = DIVIDE([Horas_Preparacion_Total], [Horas_Produccion_Total] + [Horas_Parada_Total] + [Horas_Mantenimiento_Total] + [Horas_Preparacion_Total], 0)
```

---

## 🛁 Instalación en la Planta

### 🧱 Lo que debe hacer el equipo de IT o programadores:

1. **Ejecutar las vistas** en el entorno SQL Server de producción:

   * `vista_ConCubo_Medoro7_Limpia`
   * `vista_MedoroResumen7_Final_2025`

2. **Verificar que Power BI se conecte a:** `vista_MedoroResumen7_Final_2025`

3. **Revisar que el campo `OP` en `TablaVinculadaUNION` no contenga errores graves.**

### 🔗 Conexión Power BI

* Usar la vista `vista_MedoroResumen7_Final_2025` como fuente principal.
* Importar en modo `Import` (no `DirectQuery`, salvo que sea necesario).
* Crear relaciones si hay otras tablas como calendario.

### 💪 Beneficio para la empresa

* Datos validados contra controles manuales.
* 100% reutilizable.
* Compatible con cualquier otro reporte futuro.

---

## 🔀 Estado actual del proyecto

* [x] Vistas limpias creadas
* [x] Datos validados con José (ej: OT 14620)
* [x] Saccod1 conectado correctamente
* [x] Medidas DAX definidas
* [ ] Dashboards Power BI en construcción

---

✅ **Próximos pasos**

* Crear visualizaciones Power BI con tarjetas y gráficos (como en Medoro 5 y 6).
* Exportar versión portable para José con Excel.
* Publicar video demo y documento final en GitHub.

---

© Marcelo López Castro • Proyecto Medoro 7 • Julio 2025
