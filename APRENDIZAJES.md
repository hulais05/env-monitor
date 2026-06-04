# APRENDIZAJES — env-monitor

Dashboard de monitoreo ambiental para operaciones mineras. Cliente tipo: consultora ESG que necesita entregar reportes regulatorios a autoridades de aplicación.

---

## Qué se construyó

`env-monitor/index.html` — dashboard ESG single-file con tres pestañas:

| Pestaña | Contenido |
|---------|-----------|
| Vista ejecutiva | 4 KPIs (GEI, agua, cumplimiento, inversión), gráficos de emisiones y tendencia, estado regulatorio por componente |
| Monitoreo operativo | 6 sensor cards, **mapa unificado de 4 capas** (Agua/Aire/Suelo/Relaves), balance hídrico, distribución de residuos, tabla de compliance |
| Comunidad | KPIs sociales, empleo local, incidentes abiertos, inversión comunitaria |

Funcionalidades transversales:
- Modal de detalle clickable en cada card (gráfico + tabla histórica + alertas)
- Upload de Excel para actualizar datos en el mapa (agua y aire)
- Plantillas descargables
- Responsive completo (mobile, tablet, desktop)
- Todos los gráficos con Chart.js, todos los mapas con Leaflet

---

## Decisiones de arquitectura

### Single-file HTML

**Decisión:** todo en un único `index.html` — CSS inline, JS inline, sin bundler.

**Por qué:** el cliente lo deployará en Cloudflare Pages con drag & drop. Un solo archivo elimina rutas relativas rotas, no hay build step, no hay dependencias locales. Si en el futuro escala a múltiples dashboards se modulariza.

### Stack de visualización

- **Chart.js 4.4.1** via CDN: suficiente para los gráficos de línea/barra/donut requeridos. No se necesita D3.
- **Leaflet 1.9.4** via CDN: estándar para mapas con OSM. Más liviano que Mapbox y sin API key.
- **SheetJS (xlsx)** via CDN: para leer archivos Excel del cliente sin servidor.

### Mapa unificado con 4 capas (decisión de la sesión final)

**Decisión:** reemplazar dos mapas Leaflet separados (`#waterMap` + `#airMap`) por un único mapa `#envMap` con cuatro `L.layerGroup()`.

**Por qué:**
- Dos instancias Leaflet en la misma página compiten por `invalidateSize()` en el resize y generan bugs de renderizado.
- El usuario quiere ver relaciones espaciales entre capas (ej: una alerta de suelo cerca de un pozo de agua).
- Un mapa con checkboxes es más limpio que dos scrolleando uno debajo del otro.

**Implementación:**
```js
const LAYER_CONFIG = {
  agua:    { borderColor: '#1A5A9E', points: () => MONITORING_POINTS, isAir: false },
  aire:    { borderColor: '#1578A0', points: () => AIR_MONITORING_POINTS, isAir: true },
  suelo:   { borderColor: '#5C3D1A', points: () => SOIL_MONITORING_POINTS, isAir: false },
  relaves: { borderColor: '#9E4A10', points: () => TAILING_MONITORING_POINTS, isAir: false },
};
```
- **Borde del marker** = identifica la capa (color fijo por tipo)
- **Relleno del marker** = estado operativo (verde/amarillo/rojo)
- `toggleLayer(key, visible)` llama `envMap.addLayer()` / `envMap.removeLayer()` sobre el `LayerGroup`

### Popup unificado `buildEnvPopup(pt, isAir)`

Antes había `buildPopup()` para agua y `buildAirPopup()` para aire (con manejo especial del campo "Referencia"). Se unificaron en una sola función con el flag `isAir`. Menos código, mismo resultado.

### Datos de Suelo y Relaves

Se agregaron como datos demo representativos de una minera en NOA:
- **Suelo:** pH, Pb, Zn, As en zona de acopio, flanco este, zona de revegetación
- **Relaves:** nivel freático, presión de poros, deformación, sismicidad en diques Norte (coronamiento + pie) y Sur

---

## Problemas que aparecieron y cómo se resolvieron

### Límite de tokens al escribir el archivo

**Problema:** en una sesión anterior, el archivo se cortó a mitad de escritura porque se alcanzó el límite de contexto mientras se generaba el HTML/JS completo.

**Solución:** al inicio de la siguiente sesión, se usó `/clear` para reiniciar el contexto, se leyó el archivo existente para identificar el punto de corte, y se completó la escritura desde ahí. El archivo tenía 1579 líneas — en futuras sesiones de escritura masiva, preferir edits quirúrgicos sobre rewrites completos.

**Aprendizaje:** para archivos >800 líneas, nunca usar Write completo en una sola operación. Usar Edit con `old_string`/`new_string` sobre secciones específicas.

### Dos instancias Leaflet

**Problema:** con `#waterMap` y `#airMap` como dos mapas separados, al hacer click en la pestaña "Monitoreo operativo" había que llamar `invalidateSize()` en ambos. Si uno no estaba inicializado todavía, podía lanzar error. La lógica se duplicaba.

**Solución:** mapa unificado — un solo `invalidateSize()` en `showTab()`.

### `renderEnvLayers()` no respeta checkboxes desmarcados

**Problema potencial:** cuando se carga Excel y se llama `renderEnvLayers()`, re-dibuja todos los markers incluyendo los de capas cuyo checkbox está desmarcado. Pero como `clearLayers()` vacía el LayerGroup sin quitarlo del mapa, y el LayerGroup ya estaba quitado del mapa con `removeLayer()`, los markers nuevos se agregan al LayerGroup off-map y no aparecen hasta que se reactiva el checkbox.

**Resultado:** comportamiento correcto por diseño de Leaflet — los LayerGroups desconectados del mapa no renderizan sus capas aunque se les agreguen markers.

---

## Pendiente para próximas sesiones

### Funcional urgente
- [ ] **Datos reales de Suelo y Relaves:** las coordenadas y parámetros actuales son demo. Necesitan reemplazarse con datos del proyecto real del cliente.
- [ ] **Upload de Excel para Suelo y Relaves:** actualmente solo Agua y Aire tienen botones de carga. Agregar `handleSoilExcelUpload()` y `handleTailingExcelUpload()`.
- [ ] **Plantillas XLS para Suelo y Relaves:** `downloadSoilTemplate()` y `downloadTailingTemplate()`.

### UX / mejoras
- [ ] **Zoom al activar/desactivar capa:** cuando se activa una capa, hacer `fitBounds()` sobre todos los puntos visibles.
- [ ] **Leyenda de capas en el mapa** (L.control): mostrar los swatches directamente sobre el mapa, no solo en los checkboxes.
- [ ] **Filtro de estado en checkboxes:** además de mostrar/ocultar por tipo, poder filtrar por estado (solo alertas).
- [ ] **Modal para relaves:** actualmente no hay modal de detalle dedicado para estabilidad de relaves. Agregar `'op-relaves-resumen'` en MODALS con datos históricos de los diques.

### Deploy
- [ ] **Configurar dominio** si el cliente tiene uno propio (Cloudflare Pages → Custom domain).
- [ ] **Separar datos del código:** cuando haya datos reales, moverlos a un JSON externo que se fetch al cargar. Hoy están hardcodeados en el JS.
- [ ] **Autenticación básica** si el dashboard es privado (opciones: Cloudflare Access, o password simple con localStorage).

### Técnico
- [ ] **Exportar PDF / imagen del mapa:** el cliente probablemente necesite adjuntar screenshots a sus informes regulatorios. Explorar `html2canvas` o `leaflet-easyprint`.
- [ ] **Fechas dinámicas:** los períodos "Q2 2025" y las fechas de medición están hardcodeados. Conectar con datos reales o parámetro URL `?periodo=Q2-2025`.
