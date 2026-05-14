# Brand rules — Gantt Gama Mobility

> Resumen operativo del brandbook (`proyecto-dual-write/brandbook/brandbook-agent.md`) aplicado específicamente a la generación de cartas Gantt + WBS. Si entra en conflicto con el brandbook completo, el brandbook gana.

## Colores de marca (invariables · no cambian entre claro/oscuro)

| Token CSS         | Hex       | Rol                                              |
|-------------------|-----------|--------------------------------------------------|
| `--gama-red`      | `#FF3700` | Alertas, ruta crítica, destructivas              |
| `--gama-orange`   | `#FF5F00` | CTA, links, accent principal del documento       |
| `--gama-purple`   | `#870FE6` | Categóricos, info, agrupaciones                  |
| `--gama-green`    | `#22B260` | Éxito, estados válidos, completado positivo      |

## Mapeo módulos → colores (estándar para Gantts)

Cuando un proyecto tiene 4–6 módulos, asigná en este orden:

| Posición / rol                                  | Color                | Token                 |
|-------------------------------------------------|----------------------|------------------------|
| Módulo categórico / "info" / vista interna      | Púrpura              | `var(--gama-purple)`   |
| Módulo "público positivo" / éxito / aprobación  | Verde                | `var(--gama-green)`    |
| Módulo de proceso / warning / workflow          | Naranja              | `var(--gama-orange)`   |
| Módulo crítico / integraciones riesgosas        | Rojo                 | `var(--gama-red)`      |
| Módulo "habilitador" / utilidad / neutro        | Gris medio           | `#4D4D4D` / `#B3B3B3`  |
| PM / Discovery / Gobernanza                     | Gris oscuro          | `#323232` / `#CCCCCC`  |
| QA / UAT                                        | Gris medio claro     | `#666666` / `#999999`  |

> **Por qué grises para habilitador/PM/QA:** el brandbook fija solo 4 colores de marca. Para 5+ módulos no se inventan colores — los roles transversales (PM, QA) y los habilitadores (utilidades como "maestro de correos") van en escala de grises del brandbook.

Si el módulo de ruta crítica no es de integración sino otro tipo, igualmente va en rojo — el rojo en Gantt = ruta crítica + alerta.

## Tokens semánticos por modo

```css
[data-theme="light"] {
  --bg-page:        #FAFAFA;
  --bg-surface:     #F5F5F5;
  --bg-card:        #FFFFFF;
  --bg-hover:       #EFEFEF;
  --text-primary:   #323232;
  --text-secondary: #666666;
  --text-muted:     #999999;
  --border:         #E0E0E0;
  --border-subtle:  #EEEEEE;
}

[data-theme="dark"] {
  --bg-page:        #0A0A0A;
  --bg-surface:     #1A1A1A;
  --bg-card:        #1E1E1E;
  --bg-hover:       #2A2A2A;
  --text-primary:   #FFFFFF;
  --text-secondary: #B3B3B3;
  --text-muted:     #808080;
  --border:         #333333;
  --border-subtle:  #262626;
}
```

## Spacing (base 4px)

`--space-xs:4px · --space-sm:8px · --space-md:16px · --space-lg:24px · --space-xl:32px · --space-2xl:48px · --space-3xl:64px`

**Nunca usar valores fuera de esta escala.** Si necesitás un valor intermedio, redondeá hacia arriba.

## Radius

`--radius-sm:8px (botones, inputs) · --radius-md:12px (tags pequeños) · --radius-lg:16px (cards, tablas, contenedores) · --radius-pill:9999px (badges/pills)`

## Tipografía

- **Mont** (Bold 700): H1, H2, month headers del Gantt. Si no está disponible, fallback a Montserrat Bold.
- **Montserrat** (Regular/SemiBold/Bold): todo lo demás. Cargada desde Google Fonts CDN.
- Tags y overlines siempre `text-transform: uppercase` + `letter-spacing: 2px`.

## Anti-patterns explícitos para Gantt

| ❌ NO hacer | ✅ Sí hacer |
|---|---|
| `border-left: 4px solid <color>` en cards de alcance | Borde uniforme `1px solid var(--border)` — la categoría va en el pill dentro del título |
| Pills con `border-radius: 3px` (esquinas duras) | Siempre `border-radius: 9999px` con padding `3px 12px` |
| Hex hardcodeados (`background: #FF5F00`) | Solo variables CSS (`background: var(--gama-orange)`) |
| Fondo de página `#FFFFFF` | Siempre `#FAFAFA` en claro, `#0A0A0A` en oscuro |
| Brand-bar de 6px arriba del header | Color strip Gama Full de 12px **al fondo** de la pantalla (fixed) |
| `opacity` para reducir contraste de texto secundario | Usar `var(--text-secondary)` o `var(--text-muted)` explícitos |
| Helvetica Neue / system-ui como font principal | Montserrat (body) + Mont (headings) desde Google Fonts |
| Gradientes en bars del Gantt o en cards | Colores sólidos siempre. Gama Full gradient SOLO en el color strip de 12px |
| Más de 7 KPIs/categorías por pantalla | Si hay >7 módulos, agrupá visualmente (espaciado mayor, secciones tipográficas) |
| Botón de tema flotante (position:fixed) | Toggle en el header, no overlay |

## Anatomía del Gantt aprobada

- **Grid**: `64px (WBS) | 280px (Paquete) | 56px (HH) | 96px (Owner) | repeat(N_weeks, 1fr)` donde `N_weeks` es el total de semanas del proyecto.
- **Month headers** (row 1, ocupando 4 columnas iniciales vacías + spans por mes): bg `var(--month-header-bg)` (`#323232` claro / `#252525` oscuro), texto `Mont 11px uppercase letter-spacing:2px`.
- **Week headers** (row 2): bg `var(--bg-surface)`, texto `10px Montserrat Bold uppercase`.
- **Bars**: `border-radius: 4px`, alto `16px` dentro de celda de `32px` (top/bottom `8px`), texto blanco `9px Montserrat 600 letter-spacing:0.3px`.
- **Row-phase** (separador de módulo): `grid-column: 1/-1`, bg `var(--bg-surface)`, padding `8px 16px`, texto `10px uppercase 2px letter-spacing` en el color del módulo correspondiente.
- **Critical bars** (ruta crítica): la clase `.critical` es **aditiva** — solo agrega un outline rojo (`box-shadow: 0 0 0 2px var(--bg-card) inset, 0 0 0 3px var(--gama-red)`). **NO reemplaza el background del bar**: el color del módulo (púrpura, naranja, verde, etc.) se conserva debajo. Esto permite que un bar diga "soy del módulo M2 (naranja) Y estoy en la ruta crítica (outline rojo)" simultáneamente.

## Header del documento

- Logo Gama Mobility (variante `hz-negocios` para portadas/decks). Dos `<img>` con `data-variant="light|dark"`, swap automático por CSS según `[data-theme]`.
- Toggle de tema (pill, 10px font, padding `6px 12px`) en `.header-right` arriba del bloque meta.
- Bloque meta a la derecha: `Cliente · <strong>...</strong>` / `PM Lead · ...` / `Versión · v[X.Y] · YYYY-MM-DD`.
- Subtítulo en `--gama-orange` con `text-transform: uppercase, letter-spacing: 2px`.

## Color strip Gama Full (obligatorio)

Siempre al fondo de la pantalla, fixed:

```html
<div class="color-strip" aria-hidden="true">
  <span style="background:#FF3700"></span>
  <span style="background:#FF5F00"></span>
  <span style="background:#870FE6"></span>
  <span style="background:#22B260"></span>
</div>
```

```css
.color-strip {
  position: fixed; bottom: 0; left: 0; right: 0;
  height: 12px; display: flex; z-index: 100;
}
.color-strip span { flex: 1; }
```
