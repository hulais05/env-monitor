# env-monitor

Dashboard de monitoreo ambiental ESG para operaciones mineras.
Diseñado para consultoras ESG que entregan reportes regulatorios a autoridades de aplicación.

**Live:** `https://env-monitor-9kj.pages.dev/`

---

## Funcionalidades

### Pestañas

| Pestaña | Contenido |
|---------|-----------|
| Vista ejecutiva | KPIs de emisiones GEI, agua recirculada, cumplimiento regulatorio e inversión ambiental |
| Monitoreo operativo | Sensor cards de calidad de agua, mapa interactivo 4 capas, tabla comparativa de pozos |
| Comunidad | Empleo local, incidentes ambientales, inversión social |

### Monitoreo operativo — detalle

- **6 sensor cards:** consumo hídrico, PM10, agua recirculada, pH, OD, Arsénico Total
- **Mapa unificado Leaflet** con 4 capas activables via checkbox:
  - Agua — 13 pozos de monitoreo
  - Aire — 5 estaciones de calidad del aire
  - Suelo — 3 puntos de muestreo
  - Relaves — 3 puntos de estabilidad de diques
- **Tabla comparativa** — 13 pozos × 5 parámetros con semáforo de colores por estado regulatorio
- **Exportar XLS** — descarga la tabla comparativa con columna de excedencias
- **Upload XLS** — reemplaza datos de agua o aire desde planilla propia
- **Modales de detalle** — gráfico + tabla + alertas al clickear cualquier card

---

## Stack

| Componente | Tecnología |
|-----------|------------|
| UI | HTML5 + CSS (inline, sin framework) |
| Gráficos | Chart.js 4.4.1 via CDN |
| Mapas | Leaflet 1.9.4 + OpenStreetMap |
| Excel | SheetJS (xlsx) via CDN |
| Hosting | Cloudflare Pages |

**Sin bundler. Sin build step. Sin dependencias locales.**

---

## Estructura de archivos

```
env-monitor/
├── index.html          — dashboard completo (single-file)
├── demo-data.json      — datos de 13 pozos de monitoreo (demo anonimizado)
├── DATABASE_SCHEMA.md  — esquema PostgreSQL para backend futuro (FastAPI + React)
├── APRENDIZAJES.md     — decisiones de arquitectura y lecciones aprendidas
└── Documentos/         — PDFs de referencia (propuesta técnica, estadísticas)
```

---

## Datos demo

`demo-data.json` contiene datos genéricos anonimizados basados en estadísticas reales
de monitoreo de aguas subterráneas en Puna, NOA (Proyecto Jama, Salar de Jama).

### Estructura del JSON

```json
{
  "meta": { "proyecto", "operador", "campana", "n_pozos", "n_parametros" },
  "kpi":  { "pozos_ok", "pozos_atencion", "pozos_alerta", "label_periodo" },
  "puntos_agua": [
    {
      "id": "DEMO_01",
      "name": "Pozo Demo 01",
      "type": "Piezómetro — brine",
      "lat": -23.3005, "lng": -67.0131,
      "desc": "Concesión Norte A · 16 campañas",
      "fecha": "05/12/2024",
      "params": [
        { "p": "pH",              "v": 6.46,   "u": "UpH",   "lim": "6.5–8.5", "s": "danger" },
        { "p": "Oxígeno Disuelto","v": 6814,   "u": "µg/L",  "lim": "≥ 5.000", "s": "ok"     },
        { "p": "Conductividad",   "v": 191046, "u": "µS/cm", "lim": "Sin límite","s": "ok"    },
        { "p": "Arsénico Total",  "v": 1408,   "u": "µg/L",  "lim": "≤ 50",    "s": "danger" },
        { "p": "Litio Disuelto",  "v": 824000, "u": "µg/L",  "lim": "Referencia","s": "ok"   }
      ]
    }
  ]
}
```

**Semáforo:** `s` puede ser `ok` / `warning` / `danger`. Calculado según Ley N° 24585, Anexo IV.

### Resumen de los 13 pozos

| Tipo | Pozos | Estado As Total |
|------|-------|-----------------|
| Piezómetro brine | 9 | 8 en alerta (origen geogénico volcánico) |
| DDH/RW brine | 4 | 3 en alerta |
| DDH agua dulce | 2 | OK (As < 50 µg/L) |

---

## Cómo correr localmente

El dashboard usa `fetch('demo-data.json')` — requiere HTTP server, no funciona con `file://`.

```bash
cd env-monitor/
python3 -m http.server 7422
# Abrir: http://localhost:7422
```

---

## Deploy en Cloudflare Pages

1. Ir a `dash.cloudflare.com → Workers & Pages`
2. Seleccionar el proyecto `env-monitor-9kj`
3. **New deployment → Upload assets** → arrastrar la carpeta `env-monitor/`

CLI alternativo:
```bash
npx wrangler pages deploy env-monitor/ --project-name env-monitor-9kj
```

---

## Próximas funcionalidades

- [ ] KPIs de Vista Ejecutiva dinámicos desde JSON
- [ ] Ficha de detalle por pozo (modal al clickear fila de tabla comparativa)
- [ ] Tendencia temporal — gráfico de parámetros por campaña histórica
- [ ] Upload XLS para capas Suelo y Relaves
- [ ] Exportar PDF del dashboard completo
