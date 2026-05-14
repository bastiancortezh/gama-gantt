# HTML generation — algorithm reference

> Cómo transformar los datos capturados en el flujo de la skill en el HTML final usando `assets/template.html` como base.

## Pipeline general

1. **Read** `assets/template.html`.
2. Calcular **derivados** a partir de los inputs (semanas totales, meses, mapeos color→módulo).
3. **Generar fragmentos HTML** para cada sección dinámica.
4. **Sustituir** los marcadores `{{...}}` en el template.
5. **Write** el archivo final a la ruta elegida por el usuario.

## Inputs estructurados (lo que tenés tras la entrevista)

```
{
  iniciativa: "Portal de Siniestros",
  po: "Operaciones — María Pérez",
  version: "v0.1",
  fechaDoc: "2026-05-13",
  fechaInicio: "2026-06-01",
  fechaFin: "2026-08-24",
  unidad: "Ambas",                // HH | Semanas | Ambas
  sprints: ["Sprint 1", "Sprint 2", "Sprint 3"],
  modulos: [
    { codigo: "M0", nombre: "Matriz de Siniestro", color: "purple", descripcion: "..." },
    { codigo: "M1", nombre: "Seguimiento Siniestros", color: "green", descripcion: "..." },
    ...
  ],
  paquetes: [
    {
      wbs: "M1.1", nombre: "Análisis funcional", hh: 16, sem: "S2",
      owner: "Funcional", predecesor: "—",
      inicio: "2026-06-08", fin: "2026-06-14",
      sprint: "Sprint 1", modulo: "M1"
    },
    ...
  ],
  hitos: [
    { id: "H1", nombre: "Discovery firmado", semana: "S2", validacion: "..." },
    ...
  ],
  rutaCritica: ["1.2", "M4.1", "M4.2", "M4.3", "M4.5", "M4.6", "9.2", "9.4"],  // o null
  riesgos: [{ riesgo: "...", mitigacion: "..." }, ...]                          // o null
}
```

## Derivados a calcular

### Semanas totales (`N_weeks`)
```
N_weeks = ceil((fechaFin - fechaInicio) / 7 días)
```

### Meses (`N_months`) y su span en columnas
- `N_months = ceil(N_weeks / 4)`
- Cada mes = 4 columnas semana, excepto el último que toma el resto: `last_span = N_weeks - 4*(N_months-1)`.
- Si `N_weeks` no es múltiplo de 4, distribuí lo restante en el último mes-header (más limpio que prorratear).

### Mapeo `color` → token CSS

```
purple  → var(--gama-purple) / pill class "m0"
green   → var(--gama-green)  / pill class "m1"
orange  → var(--gama-orange) / pill class "m2"
neutral → var(--m3-color)    / pill class "m3"   (gris habilitador)
red     → var(--gama-red)    / pill class "m4"
pm      → var(--pm-color)    / pill class "pm"
qa      → var(--qa-color)    / pill class "qa"
```

El template ya define `.pill.m0/.m1/.../.pm/.qa` y `.bar.m0/...`. Solo asigná las clases correctas a cada elemento.

### Posición y ancho de cada bar en el Gantt

El Gantt es un `display: grid` con `repeat(N_weeks, 1fr)` columnas para las semanas. Cada paquete genera:
- **Una celda `.cell` por semana del rango total** (todas), pero solo las que están dentro del rango del paquete contienen un `<div class="bar">`.
- El bar usa `style="left: X%; right: Y%"` para ocupar el % deseado dentro de la celda semana.

**Algoritmo simplificado** (asume semanas alineadas Lun-Dom):

```
weekIndexStart = (paquete.inicio - proyecto.fechaInicio) / 7 días   // 0-based week index
weekIndexEnd   = (paquete.fin - proyecto.fechaInicio) / 7 días

Para cada week i en [0, N_weeks):
  if i < weekIndexStart or i > weekIndexEnd:
    emit <div class="cell"></div>
  elif i == weekIndexStart:
    // Bar empieza en esta semana, posiblemente no desde el día 1
    dayOffset = (paquete.inicio - inicio_de_semana_i) / 1 día      // 0-6
    leftPct = (dayOffset / 7) * 100
    if weekIndexStart == weekIndexEnd:
      // Bar empieza y termina en la misma semana
      dayEnd = (paquete.fin - inicio_de_semana_i) / 1 día           // 0-6
      rightPct = (1 - (dayEnd + 1) / 7) * 100
      emit <div class="cell"><div class="bar {color}" style="left:{leftPct}%;right:{rightPct}%">{nombre_corto}</div></div>
    else:
      emit <div class="cell"><div class="bar {color}" style="left:{leftPct}%;right:0">{nombre_corto}</div></div>
  elif i == weekIndexEnd:
    dayEnd = (paquete.fin - inicio_de_semana_i) / 1 día
    rightPct = (1 - (dayEnd + 1) / 7) * 100
    emit <div class="cell"><div class="bar {color}" style="left:0;right:{rightPct}%">{nombre_corto}</div></div>
  else:
    // semana intermedia, bar full-width
    emit <div class="cell"><div class="bar {color}" style="left:0;right:0">{nombre_corto}</div></div>
```

**Redondeo:** si las fechas no caen en límites limpios, redondeá los `%` a múltiplos de 4 o 8 para no generar números feos (`left:13.7142857%` queda mal). Acepta pérdida de ~1 día de precisión visual.

**Nombre corto del bar:** si el `nombre` del paquete no entra (>20 chars aprox), usá el `wbs` (ej: `M1.2`) en lugar del nombre. El template ya tiene `overflow: hidden; text-overflow: ellipsis` así que truncar es OK.

### Ruta crítica → clase `critical`

Si `rutaCritica` está definida, marcá todos los bars cuyos `wbs` aparecen en la lista con la clase adicional `critical`. Esto les agrega un outline rojo que destaca la cadena.

## Sección por sección — qué generar

### `{{HEAD_TITLE}}`

```
<title>Gantt + WBS · {iniciativa} · {version}</title>
```

### `{{SUB_LABEL}}`

El "kicker" arriba del H1. Por defecto: `"Plan de Proyecto"`. Si el usuario lo override, usalo.

### `{{INICIATIVA_NOMBRE}}` → `<h1>{iniciativa}</h1>`

### `{{DESC_CORTA}}`

Auto-derivar:
```
{count_modulos} módulos · {duracion_meses} meses · {N_weeks} semanas · plan WBS + Gantt + ruta crítica
```

Si `rutaCritica` no está, quitá `"+ ruta crítica"`.

### `{{META_HTML}}`

```html
<div class="meta">
  Cliente · <strong>Gama Mobility (GAMALEASING)</strong><br>
  P.O · <strong>{po}</strong><br>
  Versión · <strong>{version} · {fechaDoc}</strong>
</div>
```

### `{{CARDS_HTML}}` — cards de alcance

Para cada módulo:
```html
<div class="card">
  <h4><span class="pill {clase_color}">{codigo}</span> {nombre_corto}</h4>
  <p>{descripcion}</p>
</div>
```

Si el módulo no tiene `descripcion`, derivá una corta del nombre o pedila al usuario en el paso 4 (descripción opcional por módulo).

### `{{WBS_HTML}}` — tabla WBS

Para cada módulo, un `<tr class="wbs-section {clase_color}"><td colspan="6">{codigo} · {nombre}</td></tr>` y luego un `<tr>` por paquete del módulo. Total al final: `<tr><td class="total-cell" colspan="2">Total estimado</td><td class="total-cell">~{total_hh} HH</td><td class="total-cell" colspan="3">{contexto_equipo}</td></tr>`.

### `{{GANTT_GRID_COLS}}`

```
grid-template-columns: 64px 280px 56px 96px repeat({N_weeks}, 1fr);
```

(Inyectar en el `<style>` del template — el template usa `{{GANTT_GRID_COLS}}` como placeholder).

### `{{GANTT_HEADER_HTML}}`

Row 1 (months): 4 div vacíos + un `<div class="month-header" style="grid-column:span {weeks_in_month}">MES {i+1}</div>` por cada mes.

Row 2 (weeks): 4 `<div class="head">` para las labels (WBS, Paquete, HH, Owner) + N `<div class="head">S{i+1}</div>`.

### `{{GANTT_BODY_HTML}}`

Para cada módulo:
- Un `<div class="row-phase {clase_color}">{codigo} · {nombre}</div>` (con `grid-column: 1/-1`).
- Para cada paquete del módulo: las 4 cells fijas (`wbs`, `name`, `effort`, `owner`) + N celdas semana según el algoritmo de posicionamiento.

### `{{HITOS_HTML}}`

Si `hitos` definidos, una `<tr>` por hito. Si no, derivar uno automático por sprint (último día del sprint).

### `{{RUTA_CRITICA_HTML}}` + `{{RIESGOS_HTML}}`

Si están definidos, incluir las cards. Si no, comentá la sección o emití un `<!-- ruta crítica no definida -->` en su lugar y un `display:none` envolvente (o directamente omití la sección — más limpio).

### `{{LOGO_PATH_LIGHT}}` y `{{LOGO_PATH_DARK}}`

El plugin trae los 2 logos necesarios bundleados en `skills/gantt-creator/assets/logos/`:
- `gamamobility-logo_hz-negocios-n.png` — para light mode (texto negro)
- `gamamobility-logo_hz-negocios-b.png` — para dark mode (texto blanco)

**Opciones de resolución de path** (preguntale al usuario solo si el bundle no aplica):

1. **Por defecto (Cowork / portable)**: copiar los 2 PNGs junto al HTML generado y usar `./gama-logo-light.png` y `./gama-logo-dark.png`. Más simple, el HTML es portable.

2. **Repo Gama local (DEV/)**: si el HTML va a `DEV/<slug>.html` y el repo Gama está clonado, podés referenciar `./proyecto-dual-write/brandbook/RGB/png/gamamobility-logo_hz-negocios-n.png` (la ruta histórica). Solo funciona si el usuario tiene el monorepo.

3. **Base64 inline**: embebé el PNG como `data:image/png;base64,...`. Más pesado (~50KB cada uno) pero zero file dependencies. Útil para enviar el HTML como un solo adjunto por email.

4. **CDN privado de Gama**: si existe una URL pública del logo, usarla directamente. Requiere internet en runtime.

Antes de escribir el HTML, **copiá los 2 logos al directorio destino** si elegiste opción 1 — el `Write` del HTML no copia archivos relacionados.

El template ya tiene `onerror="this.style.display='none'"` en las `<img>` para fallar elegantemente si el path no resuelve.

## Validación final

Antes de `Write`:

- [ ] Todos los `{{...}}` fueron sustituidos (grep al string final no debería encontrar `{{`).
- [ ] El `data-theme` no aparece hardcodeado en `<html>` — lo setea el script anti-FOUC.
- [ ] El color strip está presente al final del body.
- [ ] El toggle de tema está en `.header-right`.
- [ ] Las cards NO tienen `border-left` de color.
- [ ] La cantidad de `<div class="head">S{i}</div>` == `N_weeks`.
- [ ] La suma de `span` en `month-header` == `N_weeks`.

Si alguna check falla, corregí antes de escribir.
