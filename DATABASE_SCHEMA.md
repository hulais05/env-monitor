# DATABASE_SCHEMA.md — env-monitor

Esquema relacional para el Tablero de Control Ambiental.
Basado en la propuesta técnica Dynamik (USD 24.000, PostgreSQL + FastAPI + React/Vite)
y en la estructura real del Excel de estadísticas de pozos de agua subterránea (76 parámetros, 13 pozos, ~16 campañas por pozo).

---

## Arquitectura de referencia (Dynamik)

```
PostgreSQL  ←→  FastAPI (Python)  ←→  React + Vite
                                          ↓
                                   Leaflet (mapas)
                                   Chart.js (gráficos)
                Auth: AWS Cognito
                Infra: AWS (on-demand, ~50-100 USD/mes)
```

---

## Tablas

### 1. `proyectos`

Nivel superior. Permite que el dashboard maneje múltiples proyectos mineros.

```sql
CREATE TABLE proyectos (
    id              SERIAL PRIMARY KEY,
    codigo          VARCHAR(30) UNIQUE NOT NULL,   -- ej. 'LIT-NORTE-01'
    nombre          VARCHAR(120) NOT NULL,
    descripcion     TEXT,
    activo          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

**Datos de demo:**

| id | codigo | nombre |
|----|--------|--------|
| 1 | `LIT-NORTE-01` | Proyecto Litio Norte |
| 2 | `COBRE-SUR-01` | Proyecto Cobre Sur |

---

### 2. `concesiones`

Agrupación geográfica / legal dentro de un proyecto. En el Excel real había dos concesiones con 13 pozos entre ellas.

```sql
CREATE TABLE concesiones (
    id              SERIAL PRIMARY KEY,
    proyecto_id     INTEGER REFERENCES proyectos(id),
    codigo          VARCHAR(30) UNIQUE NOT NULL,   -- ej. 'CONC-NORTE-A'
    nombre          VARCHAR(120) NOT NULL,
    area_ha         NUMERIC(10,2),                 -- hectáreas
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

**Datos de demo:**

| id | proyecto_id | codigo | nombre | area_ha |
|----|-------------|--------|--------|---------|
| 1 | 1 | `CONC-NORTE-A` | Concesión Norte A | 4850.00 |
| 2 | 1 | `CONC-NORTE-B` | Concesión Norte B | 2100.00 |

---

### 3. `puntos_monitoreo`

Pozos, piezómetros, estaciones de muestreo. Núcleo del sistema.

```sql
CREATE TABLE puntos_monitoreo (
    id              SERIAL PRIMARY KEY,
    concesion_id    INTEGER REFERENCES concesiones(id),
    codigo          VARCHAR(40) UNIQUE NOT NULL,   -- ID interno del sistema
    codigo_anterior VARCHAR(40),                   -- ID heredado / laboratorio
    nombre          VARCHAR(120) NOT NULL,

    -- Tipo de punto
    tipo            VARCHAR(30) NOT NULL           -- 'agua_subterranea' | 'agua_superficial' | 'aire' | 'suelo'
                    CHECK (tipo IN ('agua_subterranea','agua_superficial','aire','suelo')),

    -- Subtipo (en el Excel: DDH, RW, pozo de producción, etc.)
    subtipo         VARCHAR(60),

    -- Coordenadas geográficas (WGS84, para Leaflet)
    lat             NUMERIC(10,7),
    lon             NUMERIC(10,7),

    -- Coordenadas proyectadas (Gauss Kruger / POSGAR94, como en el Excel)
    gk_x            NUMERIC(12,3),
    gk_y            NUMERIC(12,3),

    activo          BOOLEAN DEFAULT TRUE,
    notas           TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_puntos_tipo ON puntos_monitoreo(tipo);
CREATE INDEX idx_puntos_concesion ON puntos_monitoreo(concesion_id);
```

**Datos de demo** (los nombres del Excel reemplazados por genéricos):

| id | codigo | nombre | tipo | lat | lon | gk_x | gk_y |
|----|--------|--------|------|-----|-----|------|------|
| 1 | `DEMO_POZO_01` | Pozo Demo 01 | agua_subterranea | -23.300520 | -67.013147 | 3394475 | 7423771 |
| 2 | `DEMO_POZO_02` | Pozo Demo 02 | agua_subterranea | -23.289718 | -66.984058 | 3395622 | 7426853 |
| 3 | `DEMO_POZO_03` | Pozo Demo 03 | agua_subterranea | -23.318750 | -67.021389 | 3392765 | 7420530 |
| 4 | `DEMO_POZO_04` | Pozo Demo 04 | agua_subterranea | -23.289800 | -66.984050 | 3395638 | 7426847 |
| 5 | `DEMO_POZO_05` | Pozo Demo 05 | agua_subterranea | -23.308120 | -67.013800 | 3394205 | 7421907 |
| 6 | `DEMO_POZO_06` | Pozo Demo 06 | agua_subterranea | -23.281430 | -66.995720 | 3396350 | 7424417 |
| 7 | `DEMO_POZO_07` | Pozo Demo 07 | agua_subterranea | -23.319300 | -67.025000 | 3394676 | 7419560 |
| 8 | `DEMO_POZO_08` | Pozo Demo 08 (DDH) | agua_subterranea | -23.317900 | -67.023100 | 3391054 | 7420626 |
| 9 | `DEMO_POZO_09` | Pozo Demo 09 (DDH/RW) | agua_subterranea | -23.301200 | -67.007500 | 3391910 | 7423395 |
| 10 | `DEMO_POZO_10` | Pozo Demo 10 (DDH/RW) | agua_subterranea | -23.289600 | -67.003800 | 3392784 | 7426805 |
| 11 | `DEMO_POZO_11W` | Pozo Demo 11W | agua_subterranea | -23.281400 | -66.995700 | 3396353 | 7424419 |
| 12 | `DEMO_POZO_12A` | Pozo Demo 12A (RW-01) | agua_subterranea | -23.335100 | -67.030200 | 3395840 | 7416973 |
| 13 | `DEMO_POZO_13` | Pozo Demo 13 (RW-02) | agua_subterranea | -23.320900 | -67.025800 | 3396020 | 7419258 |

---

### 4. `grupos_parametros`

Categorías del Excel: Características, Aniones, Metales Totales, Metales Disueltos.

```sql
CREATE TABLE grupos_parametros (
    id      SERIAL PRIMARY KEY,
    codigo  VARCHAR(30) UNIQUE NOT NULL,
    nombre  VARCHAR(80) NOT NULL,
    orden   INTEGER DEFAULT 0   -- para ordenar en UI
);
```

**Datos de demo:**

| id | codigo | nombre | orden |
|----|--------|--------|-------|
| 1 | `CARACT` | Características Físico-Químicas | 1 |
| 2 | `ANIONES` | Aniones | 2 |
| 3 | `MET_TOT` | Metales Totales | 3 |
| 4 | `MET_DIS` | Metales Disueltos | 4 |
| 5 | `MICRO` | Microbiológico | 5 |

---

### 5. `parametros`

Los 76 parámetros identificados en el Excel. Fuente de verdad de métodos analíticos y unidades.

```sql
CREATE TABLE parametros (
    id              SERIAL PRIMARY KEY,
    grupo_id        INTEGER REFERENCES grupos_parametros(id),
    nombre          VARCHAR(120) NOT NULL,
    nombre_corto    VARCHAR(40),                   -- para labels en gráficos
    metodo          VARCHAR(60),                   -- ej. 'EPA 6020 B', 'SM 4500-H B'
    unidad          VARCHAR(20),                   -- ej. 'µg/L', 'UpH', '°C'
    lcm             NUMERIC(15,4),                 -- Límite de Cuantificación del Método
    ldm             NUMERIC(15,4),                 -- Límite de Detección del Método
    activo          BOOLEAN DEFAULT TRUE,
    UNIQUE (nombre, metodo)
);

CREATE INDEX idx_param_grupo ON parametros(grupo_id);
```

**Muestra representativa de los 76 parámetros:**

| id | grupo | nombre | metodo | unidad | lcm | ldm |
|----|-------|--------|--------|--------|-----|-----|
| 1 | CARACT | pH a 25°C | SM 4500-H B | UpH | — | — |
| 2 | CARACT | Conductividad | SM 2510 B | µS/cm | — | — |
| 3 | CARACT | Temperatura | in situ | °C | — | — |
| 4 | CARACT | Oxígeno Disuelto | SM 4500-O G | µg/L | — | — |
| 5 | CARACT | Sólidos Disueltos Totales 180°C | SM 2540 C | µg/L | 10000 | — |
| 6 | CARACT | Sólidos Suspendidos Totales | SM 2540 D | µg/L | 10000 | — |
| 7 | CARACT | Alcalinidad Total | SM 2320 B | µg/L | 1000 | 300 |
| 8 | ANIONES | Cloruro | SM 4500 Cl | µg/L | 2300 | 700 |
| 9 | ANIONES | Sulfato | SM 4500 SO4 | µg/L | 10000 | — |
| 10 | ANIONES | Fluoruro | SM 4110 B | µg/L | 500 | 50 |
| 11 | ANIONES | Nitrato como N | SM 4500 NO3 B | µg/L | 2000 | — |
| 12 | ANIONES | Nitrito como N | SM 4500-NO2 B | µg/L | 40 | 4 |
| 13 | ANIONES | Cianuro Total | ASTM 7511 | µg/L | 10 | 1 |
| 14 | MET_TOT | Arsénico Total | EPA 6020 B | µg/L | 10 | 2 |
| 15 | MET_DIS | Arsénico Disuelto | EPA 6020 B | µg/L | 10 | 2 |
| 16 | MET_TOT | Litio Total | EPA 6020 B | µg/L | 10 | 3 |
| 17 | MET_DIS | Litio Disuelto | EPA 6020 B | µg/L | 10 | 3 |
| 18 | MET_TOT | Boro Total | EPA 6020 B | µg/L | 100 | 30 |
| 19 | MET_TOT | Hierro Total | EPA 6020 B | µg/L | 10 | 3 |
| 20 | MET_TOT | Manganeso Total | EPA 6020 B | µg/L | 10 | 2 |
| 21 | MET_TOT | Plomo Total | EPA 6020 B | µg/L | 10 | 2 |
| 22 | MET_TOT | Mercurio Total | EPA 7470 A | µg/L | 1 | 0.2 |
| 23 | MET_TOT | Uranio Total | EPA 6020 B | µg/L | 10 | 2 |
| … | … | *(76 parámetros en total)* | … | … | … | … |

---

### 6. `limites_legales`

Valores guía de la **Ley N° 24585** (Protección Ambiental, Anexo IV), tal como aparecen en el Excel. Seis categorías de uso del agua.

```sql
CREATE TABLE limites_legales (
    id              SERIAL PRIMARY KEY,
    parametro_id    INTEGER REFERENCES parametros(id),

    -- Categorías de la Ley N° 24585 (columnas del Excel)
    bebida_humana           NUMERIC(15,4),   -- Fuentes de agua para bebida humana
    vida_acuatica_dulce     NUMERIC(15,4),   -- Protección vida acuática agua dulce superficial
    vida_acuatica_salada    NUMERIC(15,4),   -- Protección vida acuática aguas saladas
    vida_acuatica_salobre   NUMERIC(15,4),   -- Protección vida acuática aguas salobres
    irrigacion              NUMERIC(15,4),   -- Para irrigación
    bebida_ganado           NUMERIC(15,4),   -- Para bebida de ganado

    -- 'NE' = No Establecido en la ley → NULL en la BD
    -- Campos de texto para casos especiales (rangos como '6,5-8,5')
    bebida_humana_nota      VARCHAR(30),
    irrigacion_nota         VARCHAR(30),

    fuente_normativa        VARCHAR(120) DEFAULT 'Ley N° 24585 — Anexo IV',
    vigente                 BOOLEAN DEFAULT TRUE,
    UNIQUE (parametro_id)
);
```

**Ejemplos de límites:**

| Parámetro | Bebida humana | Vida acuát. dulce | Irrigación | Bebida ganado |
|-----------|--------------|-------------------|------------|---------------|
| pH | 6,5–8,5 | 6,5–9,0 | 6,5–8,5 | 6,5–8,5 |
| Oxígeno Disuelto | 5000 µg/L | 5000 | 5000 | 5000 |
| Arsénico Total | 50 µg/L | 50 | 100 | 500 |
| Nitrato como N | 10000 µg/L | NE | NE | NE |
| Mercurio Total | 1 µg/L | 0.1 | NE | NE |
| Plomo Total | 50 µg/L | 1 | 5000 | NE |
| Boro Total | NE | 750 | 500 | 5000 |
| Litio Disuelto | NE | NE | 2500 | NE |

---

### 7. `campanas_muestreo`

Cada campaña de muestreo / protocolo de laboratorio. En el Excel: 16 campañas por pozo, con número de protocolo y fecha.

```sql
CREATE TABLE campanas_muestreo (
    id                  SERIAL PRIMARY KEY,
    proyecto_id         INTEGER REFERENCES proyectos(id),
    numero_protocolo    VARCHAR(30),               -- ej. 'MA24-00167.0012'
    fecha_muestreo      DATE NOT NULL,
    tipo_muestra        VARCHAR(40) DEFAULT 'Agua Subterránea',
    laboratorio         VARCHAR(120),
    observaciones       TEXT,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (numero_protocolo)
);

CREATE INDEX idx_campanas_fecha ON campanas_muestreo(fecha_muestreo);
```

**Datos de demo** (fechas reales del Excel, laboratorio genérico):

| id | numero_protocolo | fecha_muestreo | laboratorio |
|----|-----------------|----------------|-------------|
| 1 | `CAMP-2024-001` | 2024-02-08 | Laboratorio Analítico Demo SRL |
| 2 | `CAMP-2024-002` | 2024-04-24 | Laboratorio Analítico Demo SRL |
| 3 | `CAMP-2024-003` | 2024-05-22 | Laboratorio Analítico Demo SRL |
| 4 | `CAMP-2024-004` | 2024-07-04 | Laboratorio Analítico Demo SRL |
| 5 | `CAMP-2024-005` | 2024-07-23 | Laboratorio Analítico Demo SRL |
| 6 | `CAMP-2024-006` | 2024-08-29 | Laboratorio Analítico Demo SRL |
| 7 | `CAMP-2024-007` | 2024-09-25 | Laboratorio Analítico Demo SRL |
| 8 | `CAMP-2024-008` | 2024-10-23 | Laboratorio Analítico Demo SRL |
| 9 | `CAMP-2024-009` | 2024-11-21 | Laboratorio Analítico Demo SRL |
| 10 | `CAMP-2024-010` | 2024-12-05 | Laboratorio Analítico Demo SRL |

---

### 8. `mediciones`

Tabla central. Una fila = un valor analítico de un parámetro, en un punto, en una campaña.

```sql
CREATE TABLE mediciones (
    id                  BIGSERIAL PRIMARY KEY,
    campana_id          INTEGER REFERENCES campanas_muestreo(id),
    punto_id            INTEGER REFERENCES puntos_monitoreo(id),
    parametro_id        INTEGER REFERENCES parametros(id),

    -- Valor
    valor_raw           VARCHAR(30),               -- texto original del lab: '848', '<2', 'NR', 'NE'
    valor_numerico      NUMERIC(18,6),             -- NULL si es '<LDM' o 'NR'
    bajo_deteccion      BOOLEAN DEFAULT FALSE,      -- TRUE cuando valor_raw empieza con '<'
    no_reportado        BOOLEAN DEFAULT FALSE,      -- TRUE cuando valor_raw = 'NR'

    -- Semáforo (calculado al insertar / recalculable)
    -- Compara valor_numerico contra el límite de bebida_humana por defecto
    semaforo            VARCHAR(10) DEFAULT 'GRIS'
                        CHECK (semaforo IN ('VERDE','AMARILLO','ROJO','GRIS')),

    -- Es muestra duplicada (QA/QC)
    es_duplicado        BOOLEAN DEFAULT FALSE,

    notas               TEXT,
    created_at          TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE (campana_id, punto_id, parametro_id, es_duplicado)
);

CREATE INDEX idx_med_punto     ON mediciones(punto_id);
CREATE INDEX idx_med_parametro ON mediciones(parametro_id);
CREATE INDEX idx_med_campana   ON mediciones(campana_id);
CREATE INDEX idx_med_semaforo  ON mediciones(semaforo);
```

---

### 9. `estadisticas_punto_parametro`

Estadísticas precalculadas por pozo + parámetro (como la hoja "Resumen Resultados" del Excel). Se regenera al insertar nuevas mediciones.

```sql
CREATE TABLE estadisticas_punto_parametro (
    id                  SERIAL PRIMARY KEY,
    punto_id            INTEGER REFERENCES puntos_monitoreo(id),
    parametro_id        INTEGER REFERENCES parametros(id),

    n_observaciones     INTEGER,
    n_bajo_deteccion    INTEGER,
    promedio            NUMERIC(18,6),
    mediana             NUMERIC(18,6),
    desv_estandar       NUMERIC(18,6),
    valor_minimo        NUMERIC(18,6),
    valor_maximo        NUMERIC(18,6),

    fecha_primera       DATE,
    fecha_ultima        DATE,

    -- Semáforo agregado: peor estado registrado
    semaforo_max        VARCHAR(10) CHECK (semaforo_max IN ('VERDE','AMARILLO','ROJO','GRIS')),

    updated_at          TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (punto_id, parametro_id)
);
```

---

## Lógica del semáforo

La lógica se aplica en la capa de aplicación (FastAPI) al insertar mediciones,
comparando `valor_numerico` contra `limites_legales.bebida_humana` (uso por defecto).

```python
def calcular_semaforo(valor: float | None, limite: float | None) -> str:
    if valor is None:
        return "GRIS"          # bajo detección o no reportado
    if limite is None:
        return "GRIS"          # parámetro sin límite definido (NE)
    if valor > limite:
        return "ROJO"
    if valor > limite * 0.80:  # ≥80% del límite → alerta temprana
        return "AMARILLO"
    return "VERDE"
```

El frontend puede recalcular el semáforo en tiempo real si el usuario cambia
la categoría de uso (bebida humana → irrigación → ganado).

---

## Diagrama de relaciones

```
proyectos
    └── concesiones
            └── puntos_monitoreo
                    └── mediciones ──── campanas_muestreo
                            │                └── proyectos
                            ├── parametros ── grupos_parametros
                            │       └── limites_legales
                            └── estadisticas_punto_parametro
```

---

## Notas de implementación (Dynamik → nuestro stack)

| Dynamik | env-monitor |
|---------|-------------|
| PostgreSQL | SQLite en dev / PostgreSQL en prod (Neon o Supabase free tier) |
| FastAPI | FastAPI o endpoints serverless en Cloudflare Workers |
| React + Vite | Vanilla JS + Chart.js + Leaflet (stack actual) |
| AWS Cognito | Auth simple con JWT si escala; sin auth en MVP |
| AWS infra | Cloudflare Pages (gratis para el dashboard estático) |

**El esquema es intencionalmente compatible con PostgreSQL** para que pueda migrarse
tal cual si el proyecto escala a la arquitectura Dynamik sin cambios en la BD.

---

## Volumen estimado

Con datos reales del Proyecto Demo:
- 13 pozos × 76 parámetros × 16 campañas = ~15.800 mediciones
- A razón de 10 campañas/año → ~10.000 mediciones nuevas/año
- Tamaño estimado: < 50 MB en PostgreSQL para 5 años de datos

*Última actualización: 2026-06-03*
