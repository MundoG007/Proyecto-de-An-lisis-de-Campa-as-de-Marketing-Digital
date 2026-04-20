# Proyecto de Análisis de Campañas de Marketing Digital

**Sistema end-to-end de reporting para una agencia de marketing digital — del modelo dimensional en PostgreSQL hasta 3 dashboards ejecutivos en Power BI.**

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-13+-316192?logo=postgresql&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?logo=powerbi&logoColor=black)
![SQL](https://img.shields.io/badge/SQL-Analytical-4479A1?logo=postgresql&logoColor=white)
![Status](https://img.shields.io/badge/status-completed-success)

> ⚠️ **Nota sobre los datos:** Todos los datos del proyecto son **sintéticos y simulados** — generados específicamente para este portfolio. Los nombres de clientes, montos y métricas no corresponden a ninguna entidad real.

---

## 📋 Tabla de Contenidos

1. [Resumen Ejecutivo](#-resumen-ejecutivo)
2. [Contexto de Negocio](#-contexto-de-negocio)
3. [Arquitectura de la Solución](#-arquitectura-de-la-solución)
4. [Modelo de Datos](#-modelo-de-datos)
5. [Capa Analítica (Vistas SQL)](#-capa-analítica-vistas-sql)
6. [Los 3 Reportes de Power BI](#-los-3-reportes-de-power-bi)
7. [Decisiones Técnicas Destacadas](#-decisiones-técnicas-destacadas)
8. [Qué Demuestra este Proyecto](#-qué-demuestra-este-proyecto)
9. [Posibles Extensiones](#-posibles-extensiones)
10. [Estructura del Repositorio](#-estructura-del-repositorio)

---

## 🎯 Resumen Ejecutivo

Este proyecto simula el sistema de reporting que necesitaría una **agencia de marketing digital** que gestiona campañas pagadas (Meta, Google Ads, TikTok) para una cartera de ~74 clientes. Resuelve tres preguntas de negocio diferentes con **una sola fuente de verdad**:

- **¿Qué tan bien estamos ejecutando las campañas?** (reporte comercial)
- **¿Qué clientes son realmente rentables para la agencia?** (reporte interno de cartera)
- **¿Cómo está performando la inversión de cada cliente individual?** (reporte por cliente)

**Stack técnico:** PostgreSQL 13+ (modelado dimensional, vistas analíticas), SQL (lógica de negocio y KPIs), Power BI (visualización).

**Lo que el proyecto demuestra:** modelado dimensional limpio, diseño de capa semántica reutilizable, definición rigurosa de métricas de marketing (ROAS, CTR, CPL, CPA, margen), y traducción de datos a narrativas accionables para tres audiencias distintas.

---

## 🏢 Contexto de Negocio

Una agencia de marketing digital típica enfrenta un problema de **reporting fragmentado**: los equipos comerciales quieren ver qué campañas ganan, dirección quiere saber qué clientes dejan margen, y cada cliente quiere su propio drill-down. En la práctica esto suele resolverse con planillas de Excel desconectadas y cálculos inconsistentes entre equipos.

Este proyecto construye una **capa analítica única** que alimenta los tres casos de uso sin duplicar lógica:

| Audiencia | Reporte | Preguntas que responde |
|-----------|---------|------------------------|
| Equipo comercial / dirección | **Comercial** | ¿Qué canales y creatividades convierten mejor? ¿Qué clientes generan más ROAS? |
| Dirección / finanzas | **Interno** | ¿Qué clientes son rentables? ¿Quién está en riesgo (margen negativo, pago atrasado)? |
| Account managers / clientes | **Cliente** | ¿Cómo va mi inversión? ¿Qué ads funcionan? ¿Qué audiencias convierten? |

---

## 🏗️ Arquitectura de la Solución

```
┌─────────────────────┐       ┌──────────────────────┐       ┌──────────────────┐
│  Capa Transaccional │       │   Capa Analítica     │       │  Capa de Consumo │
│  (schemas dim/fct)  │  ──▶  │  (schema analytics)  │  ──▶  │    (Power BI)    │
│                     │       │                      │       │                  │
│  Tablas normalizadas│       │  5 vistas con KPIs   │       │  3 dashboards    │
│  Grano diario       │       │  pre-calculados      │       │  interactivos    │
└─────────────────────┘       └──────────────────────┘       └──────────────────┘
         ↑                              ↑
    Datos sintéticos              Lógica de negocio
    insertados                    centralizada
```

**Principio de diseño:** Power BI consume **vistas**, no tablas crudas. Esto centraliza las reglas de negocio (cómo se calcula ROAS, qué significa "cliente en riesgo") en SQL, donde pueden versionarse, testearse y auditarse — evitando que la lógica quede atrapada en DAX de cada reporte.

---

## 🗂️ Modelo de Datos

El modelo sigue un **esquema estrella** clásico, dividido en tres schemas:

### `dim` — Dimensiones
- `clientes` — cartera de la agencia (industria, país, moneda, fecha de alta, estado)
- `campanas` — campañas publicitarias (plataforma, objetivo, estado)
- `ads` — anuncios individuales (formato, tema creativo)
- `audiencias` — segmentaciones (tipo de audiencia, nivel de intención)
- `fecha` — dimensión calendario (año, mes, trimestre)

### `fct` — Hechos
- `metricas_ads_diarias` — grano **fecha × ad**, con impresiones, clicks, gasto, leads, conversiones, ingresos atribuidos
- `horas_agencia_diarias` — grano **fecha × cliente**, con horas internas y costo interno
- `facturacion_mensual` — grano **mes × cliente**, con tipo de contrato y monto facturado

### `analytics` — Capa semántica
Vistas reutilizables que combinan las anteriores. Ver sección siguiente.

**¿Por qué tres tablas de hechos en vez de una?** Porque tienen **granos distintos**. Intentar unificarlas fuerza joins artificiales y corrompe los cálculos de margen (el gasto en ads es diario; la facturación es mensual). Separarlas y unirlas en la capa analítica es más limpio y más honesto con la realidad del negocio.

---

## 🔍 Capa Analítica (Vistas SQL)

Cinco vistas resuelven las necesidades de los tres reportes. Cada una tiene un **grano explícito** y una responsabilidad clara:

| Vista | Grano | Propósito |
|-------|-------|-----------|
| `v_metricas_diarias` | fecha × ad | Base enriquecida con todas las dimensiones — permite filtrar en Power BI por cualquier atributo |
| `v_resumen_mensual_cliente` | mes × cliente | Une gasto en ads + costo interno + facturación → calcula **margen** y **margen %** |
| `v_performance_campana` | campaña (acumulado) | Ranking de campañas por ROAS, ingresos, leads |
| `v_performance_ad` | ad (acumulado) | Ranking de creatividades — qué formato y tema creativo gana |
| `v_alertas_cartera` | mes × cliente | Flags booleanos: margen negativo, ROAS bajo, costo alto, pago tardío |

### Ejemplo: cómo se calcula el margen

```sql
-- Extracto de v_resumen_mensual_cliente
COALESCE(fm.monto_total, 0) - COALESCE(hrs.costo_interno, 0) AS margen,
CASE 
    WHEN COALESCE(fm.monto_total, 0) = 0 THEN NULL
    ELSE (COALESCE(fm.monto_total, 0) - COALESCE(hrs.costo_interno, 0))::NUMERIC 
         / fm.monto_total
END AS margen_pct
```

**Dos decisiones visibles aquí:**
1. `COALESCE` para tratar clientes sin facturación o sin horas registradas como 0, no como NULL — evita que el margen desaparezca del reporte.
2. `NULLIF` implícito via `CASE WHEN ... = 0 THEN NULL` para **ratios**, evitando divisiones por cero. Esta regla aplica a todos los KPIs derivados (CTR, CPC, CPL, CPA, ROAS, margen %).

---

## 📊 Los 3 Reportes de Power BI

### 1. Reporte Comercial — *¿Qué está funcionando?*

![Reporte Comercial](<img width="1237" height="735" alt="image" src="https://github.com/user-attachments/assets/133f8053-630d-40f7-ad66-93f8698e3284" />
)

Vista ejecutiva del negocio: KPIs globales (Gasto, Leads, Ingresos, ROAS), desempeño por plataforma (CPL y ROAS), ROAS por cliente, ranking de creatividades ganadoras, y evolución mensual de gasto vs. ingresos.

**Preguntas que responde:**
- ¿Qué plataforma convierte mejor por coste? (CPL por plataforma)
- ¿Qué plataforma genera más retorno? (ROAS por plataforma)
- ¿Qué formato × tema creativo performa mejor? (tabla de creatividades ganadoras)
- ¿Qué clientes son los que más ROAS generan?

### 2. Reporte Interno — *¿Qué clientes son rentables?*

![Reporte Interno]([docs/reporte_interno.png](https://github.com/MundoG007/Proyecto-de-An-lisis-de-Campa-as-de-Marketing-Digital/blob/main/reporte_interno.png?raw=true))

Vista de cartera para dirección: facturación, costo interno, margen, margen %, clientes activos vs. churn vs. pausados, modelo de cobro más rentable (% de ganancias / mensual fijo / pago por lead), rentabilidad por antigüedad del cliente, y detalle cliente-mes con estado de pago.

**Preguntas que responde:**
- ¿Qué modelo de cobro deja más margen?
- ¿Los clientes antiguos son más rentables que los nuevos?
- ¿Qué clientes están en **riesgo** (margen negativo, ROAS bajo, costo alto, pago atrasado)?
- ¿Cuánto estamos facturando vs. cuánto cuesta servir la cartera?

### 3. Reporte Cliente — *¿Cómo voy?*

![Reporte Cliente](docs/reporte_cliente.png)

Drill-down individual: KPIs del cliente (gasto, leads, conversiones, ingresos, ROAS, CPL), tendencia diaria gasto vs. ingresos, embudo de conversión (impresiones → clicks → leads → conversiones), estado de campañas activas/pausadas, top creatividades del cliente, e impacto por tipo de audiencia.

**Preguntas que responde:**
- ¿Qué campañas están activas, pausadas o finalizadas?
- ¿Qué creatividades y audiencias están funcionando mejor?
- ¿Dónde se cae el embudo?

---

## 🛠️ Decisiones Técnicas Destacadas

Estas son las decisiones que un entrevistador técnico probablemente preguntaría:

### 1. Vistas analíticas, no tablas materializadas
**Por qué:** los volúmenes simulados permiten que las vistas corran rápido y se recalculen en tiempo real. En producción, las vistas más pesadas (`v_resumen_mensual_cliente`) se convertirían en **vistas materializadas con refresh nocturno**, manteniendo la misma interfaz para Power BI.

### 2. Manejo explícito de divisiones por cero
Todos los ratios usan `CASE WHEN denominador = 0 THEN NULL` (o `NULLIF`). Devolver `NULL` en vez de `0` o romper la query es la decisión correcta: **NULL es honesto — significa "no calculable"**, mientras que 0 es una mentira (implicaría que el ROAS fue cero, no que no hubo gasto).

### 3. Flags booleanos en la vista de alertas
En vez de dejar que cada reporte defina "cliente en riesgo" de forma inconsistente, `v_alertas_cartera` materializa cuatro flags: `flag_margen_negativo`, `flag_roas_bajo`, `flag_costo_alto`, `flag_pago_tardio`. Los umbrales viven en un solo lugar (SQL), no duplicados en cada DAX.

### 4. Grano explícito y documentado en cada vista
Cada vista declara su grano en el comentario (`grano: mes - cliente`). Esto previene el error clásico de Power BI: sumar métricas que no se pueden sumar porque están a granos distintos.

### 5. Schema `app.id_cliente` para testing manual
El reporte 3 usa `SET app.id_cliente TO '1'` + `current_setting('app.id_cliente')::INTEGER` — un patrón de PostgreSQL que permite parametrizar queries sin prepared statements, útil para validar en psql/pgAdmin antes de conectar Power BI.

---

## 💼 Qué Demuestra este Proyecto

| Skill | Evidencia en el proyecto |
|-------|--------------------------|
| **Modelado dimensional** | Schema estrella con 3 tablas de hechos a granos distintos, 5 dimensiones |
| **SQL analítico** | CTEs, window functions implícitas en agregaciones, joins complejos, manejo de NULLs |
| **Diseño de capa semántica** | 5 vistas reutilizables que separan lógica de negocio de visualización |
| **Definición de KPIs de marketing** | ROAS, CTR, CPC, CPL, CPA, margen, margen %, costo_interno_ratio — todos con denominadores protegidos |
| **Power BI** | 3 dashboards, cada uno optimizado para una audiencia distinta |
| **Pensamiento de producto** | Un mismo modelo sirve a tres casos de uso sin duplicar lógica |
| **Documentación** | Comentarios en SQL, README estructurado, diagramas de arquitectura |

---

## 🚀 Posibles Extensiones

Lo que agregaría con más tiempo — y que mencionaría en una entrevista si preguntan "¿qué le falta?":

- **Tests de calidad de datos** con `dbt` o `Great Expectations` (row counts, unicidad de PKs, rangos de fechas, validación de que `gasto ≥ 0`).
- **Vistas materializadas + refresh incremental** para `v_resumen_mensual_cliente` cuando los volúmenes crezcan.
- **Atribución multi-touch** — hoy el campo `ingresos_atribuidos` es una caja negra. En producción declararía la ventana y el modelo (last-click vs. lineal vs. data-driven).
- **Detección de creative fatigue** — una vista que identifique ads con CTR decreciente en ventanas rodantes de 2–6 semanas.
- **Alertas automatizadas** — un job que corra `v_alertas_cartera` diariamente y notifique a los account managers cuando un cliente cruza un umbral de riesgo.
- **Cohortes de retención de clientes** — análisis de churn por antigüedad, industria y modelo de cobro.

---

## 📁 Estructura del Repositorio

```
proyecto-marketing/
├── README.md                           ← este archivo
├── sql/
│   ├── 01_esquema_datos.sql            ← DDL: schemas dim, fct, analytics
│   ├── 02_vistas_medidas.sql           ← 5 vistas analíticas
│   └── 03_consultas_reportes.sql       ← queries que alimentan los 3 reportes
├── powerbi/
│   └── proyecto_campaña_marketing.pbix ← archivo Power BI
└── docs/
    ├── reporte_comercial.png           ← screenshot dashboard 1
    ├── reporte_interno.png             ← screenshot dashboard 2
    └── reporte_cliente.png             ← screenshot dashboard 3
```

---

## 📬 Contacto

Si el proyecto te interesa o quieres hablar de cómo se podría adaptar a otro negocio, escríbeme.

**Tecnologías:** PostgreSQL · SQL · Power BI · DAX · Modelado dimensional · Storytelling con datos
