---
name: gantt-creator
description: Use whenever the user types `/gantt`, asks for "una carta gantt", "Gantt + WBS", "cronograma de proyecto", "plan de proyecto", "WBS", or otherwise wants a project timeline document for a Gama Mobility initiative. Drives a strict interactive flow that captures initiative metadata (Nombre, P.O, Versión — siempre obligatorios, no skip), sprints/hitos, módulos with HH/Semanas/Ambas, owners, predecesores y fechas; persists the structured data to Notion under a "Gantts" parent page (one subpage per iniciativa, one page per versión); detects existing initiatives to allow iterative version bumps; and generates a branded HTML Gantt + WBS following the Gama Mobility brandbook (tokens CSS, color strip Gama Full, modo claro + oscuro, sin border-left coloreado en cards). Activate even if the user phrases it casually ("armame el gantt", "necesito un cronograma para X"). Source of truth: Gama Mobility Design System.
---

# Gantt Creator · Gama Mobility

> **Misión:** convertir una conversación de ~5–10 minutos con el PM en un Gantt + WBS marca Gama, persistido en Notion para trazabilidad de versiones y exportado a HTML standalone listo para presentar.

Esta skill se invoca por el comando `/gantt` del mismo plugin, o cuando el usuario pide algo equivalente en lenguaje natural.

---

## El contrato (no-negociable)

1. **Las 3 preguntas obligatorias del inicio** son: **Nombre de iniciativa**, **Product Owner**, **Versión**. Nunca las omitas, nunca las inferas, nunca las "deduces del contexto". Si el usuario intenta saltarlas, explicale que son obligatorias por contrato del comando y volvé a preguntarlas.
2. **Persistir en Notion antes de generar HTML**. Estructura: página padre `Gantts` → subpágina `[Nombre de iniciativa]` → subpágina `v[X.Y]`. Si `Gantts` no existe, créala. Si la iniciativa no existe, créala. Cada versión es una página propia bajo la iniciativa (histórico claro, no se sobreescribe).
3. **Detectar iniciativa existente y permitir iterar**. Si en la búsqueda Notion encontrás una subpágina bajo `Gantts` cuyo nombre coincide (fuzzy) con el de la iniciativa, ofrecele al usuario: (a) cargar la última versión como base y solo capturar deltas, o (b) partir de cero como versión nueva. La versión propuesta por defecto incrementa el minor (v0.1 → v0.2 → v0.3).
4. **El HTML final debe seguir el brandbook Gama** — ver `references/brand-rules.md`. Usa el template embebido en `assets/template.html` como base estructural.
5. **Preguntá la ruta de salida del HTML al final**, después de validar la estructura. No escribas archivos antes de tener el OK del usuario sobre dónde.

---

## El flujo, paso por paso

### Paso 0 · Detectar contexto previo

Si el usuario pasó argumentos a `/gantt` (`$ARGUMENTS`), tomalo como pista del nombre de iniciativa y saltá directo al **Paso 2** (búsqueda en Notion) usando ese valor como hipótesis. De lo contrario, arrancá en el Paso 1.

### Paso 1 · Las 3 preguntas obligatorias

Usá **una sola** invocación de `AskUserQuestion` con `multiSelect: false` para las tres preguntas. No las disperses en mensajes separados — el usuario debe verlas juntas para entender que son el bloque inicial.

Como `AskUserQuestion` requiere opciones predefinidas y aquí necesitamos texto libre, hacelo así: usá `AskUserQuestion` para las opciones-pista (ej. versiones típicas) **solo si** ayuda; para campos puramente texto libre como nombre y P.O, hacé la pregunta como texto directo en tu mensaje y esperá la respuesta del usuario. Patrón sugerido:

```
Antes de armar el Gantt necesito 3 datos obligatorios:

1. **Nombre de la iniciativa** (ej: "Portal de Siniestros", "Migración ERP")
2. **Product Owner** (nombre o área, ej: "Operaciones — María Pérez")
3. **Versión** (ej: v0.1 si es nueva; si es iteración, incrementá)

Respondeme los 3 en una línea separada cada uno.
```

Validá: si falta alguno, repetí la pregunta solo por el faltante.

### Paso 2 · Buscar/crear en Notion

Lee `references/notion-flow.md` para el detalle de tools y queries. Resumen:

1. **Encontrar/crear `Gantts`**: usá `notion-search` para buscar `"Gantts"` con `query_type: "internal"`. Si existe una página llamada exactamente `Gantts` y está en el workspace raíz (no es subpágina de otra cosa), usá esa. Si no existe, creala con `notion-create-pages` sin parent (root).
2. **Encontrar/crear la subpágina de iniciativa**: busca con el nombre dado bajo `Gantts`. Si hay match exacto, conservalo; si hay match fuzzy (caps/espacios/typos), preguntá al usuario si es la misma; si no hay match, creala como hijo de `Gantts`.
3. **Si la iniciativa ya tenía versiones**, listá las versiones existentes (subpáginas con nombre `v*`) y proponé incremento. Ejemplo: existe `v0.1, v0.2` → proponer `v0.3`. Confirmá la versión con el usuario antes de avanzar.
4. **Si el usuario eligió iterar desde la anterior**, leé la última versión con `notion-fetch` y mostrale un resumen de qué hay (sprints, módulos) para que diga qué quiere cambiar. Si eligió partir de cero, ignorá la anterior.

> **Importante:** en este paso aún NO crees la subpágina de la nueva versión. Solo la creás al final, con los datos completos. Mantené los IDs/URLs de `Gantts` y de la subpágina de iniciativa en memoria de la conversación.

### Paso 3 · Estructura del proyecto

Ahora capturá la estructura macro. Una sola tanda con `AskUserQuestion`:

- **Cantidad de Sprints/Hitos** (número entero, sugerí opciones: 2, 3, 4, 6, 8, otro)
- **Unidad de tiempo**: `HH` (horas-hombre), `Semanas`, o `Ambas` (la tabla WBS mostrará HH + columna de semanas, el Gantt usa semanas para los bloques)
- **Fecha de inicio del proyecto** (texto libre — ISO `YYYY-MM-DD` preferido; si no, normalizalo)
- **Fecha de término del proyecto** (idem)

Calculá la cantidad total de semanas entre inicio y término. Validá que cuadre con el rango "cantidad de Sprints × duración promedio". Si no cuadra, advertí pero no bloquees.

### Paso 4 · Módulos / paquetes de trabajo

Este es el paso largo. La mejor UX es **pedirle al usuario la lista completa en formato tabular**, no preguntar módulo por módulo (sería tedioso). Patrón:

```
Pasame todos los módulos/paquetes en formato CSV-like (uno por línea):

CODIGO | NOMBRE | HH | SEMANAS | OWNER | PREDECESOR | FECHA_INICIO | FECHA_FIN | SPRINT

Ejemplo:
M1.1 | Análisis funcional | 16 | S2 | Funcional | — | 2026-06-01 | 2026-06-07 | Sprint 1
M1.2 | Backend CRUD | 40 | S3-S4 | Backend | M1.1 | 2026-06-08 | 2026-06-21 | Sprint 1

Si trabajamos solo en HH, deja SEMANAS vacío (lo calculo).
Si trabajamos solo en SEMANAS, deja HH vacío.
Si la unidad es Ambas, completá las dos.
SPRINT es el agrupador (Sprint 1, Sprint 2, Hito 1, etc.) — debe coincidir con la cantidad declarada en el paso anterior.
```

Aceptá pegado de tabla Excel/Markdown también — detectá el delimitador (`|`, `\t`, `,`). Si el usuario provee menos campos de los pedidos, preguntá por los faltantes solo de las filas afectadas.

**Validaciones críticas antes de avanzar:**
- Cada módulo tiene CODIGO único.
- Cada predecesor referencia un CODIGO que existe en la lista (o `—` / vacío).
- Las fechas caen dentro del rango del proyecto.
- Cada SPRINT declarado tiene al menos 1 módulo.
- Si hay dependencias circulares, advertí y pedí corregir.

### Paso 5 · Hitos firmables (opcional pero recomendado)

Preguntá si el usuario quiere agregar hitos firmables explícitos (puntos de control con firma del cliente). Si dice sí, pedí: `H# | Hito | Semana | Cómo se valida`. Si dice no, derivá hitos básicos automáticamente (uno al final de cada Sprint).

### Paso 6 · Ruta crítica + riesgos (opcional)

Preguntá si quiere agregar:
- **Ruta crítica** (texto en formato `A → B → C → ...`)
- **Riesgos prioritarios** (lista de `Riesgo | Mitigación`)

Si dice no, omití esas secciones del HTML.

### Paso 7 · Generar el HTML

Lee `assets/template.html`. Es un template completo con marcadores `{{...}}` para los datos dinámicos. Sustituí los marcadores con los datos capturados. Las reglas de generación dinámica (especialmente las celdas del Gantt) están en `references/html-generation.md`.

**Reglas no-negociables del HTML** (también detalladas en `references/brand-rules.md`):
- `data-theme` se resuelve por script en `<head>` (claro/oscuro/system). No hardcodees `light`.
- Color strip Gama Full (12px) fijo al fondo — siempre presente.
- Cards **sin** `border-left` coloreado — diferenciación solo por pill dentro del título.
- Pills `border-radius: 9999px`, padding `3px 12px`, tint 12% del color de marca correspondiente.
- Tipografía: Montserrat (body) + Mont (H1–H3). Fonts cargadas desde Google Fonts.
- Spacing escala 4/8/16/24/32/48/64 — nunca valores arbitrarios.
- Logo Gama Mobility con swap automático claro/oscuro (`*-n.png` / `*-b.png` variantes).
- Toggle de tema en el header (no botón flotante).

### Paso 8 · Persistir versión en Notion

Antes de escribir el HTML al disco, creá la subpágina de versión en Notion bajo la iniciativa:

- Título: `v[X.Y]` (la versión confirmada en Paso 2)
- Contenido: bloque por sección — Metadata (Iniciativa, P.O, Versión, Fecha creación, Fechas proyecto), tabla WBS, tabla Hitos, Ruta crítica, Riesgos. Usa bloques Notion nativos (heading, table, paragraph) — no embebas el HTML completo, solo los datos estructurados.
- Al final del contenido, agregá un bloque de código con la ruta absoluta donde se guardó el HTML local, para trazabilidad.

Lee `references/notion-flow.md` para el detalle de cómo armar los bloques.

### Paso 9 · Preguntar ruta y escribir HTML

Preguntá ahora (no antes):
```
¿Dónde guardo el HTML?
A) cwd (directorio actual) → <cwd>/<slug-iniciativa>_v<X.Y>.html
B) Junto al ejemplo en DEV/ → DEV/<slug-iniciativa>_v<X.Y>.html
C) Otra ruta (pásame el path absoluto)
```

Usá `Write` para escribir el archivo. Slug del nombre: lowercase, sin acentos, espacios → `_`, sin caracteres especiales.

### Paso 10 · Cierre

Reportá al usuario:
- Ruta del HTML escrito
- URL de la página Notion de la versión
- Resumen: N módulos, N sprints, N hitos, N semanas totales
- Próxima invocación: si querés iterar, invocá `/gantt "[nombre iniciativa]"` y la skill detecta y propone v[X.Y+1]

---

## Antipatrones (no hagas esto)

| Antipatrón | Por qué no |
|---|---|
| Inferir Nombre/P.O/Versión del contexto sin preguntarlos | Rompe el contrato del comando. Esas 3 son auditables y no-modificables. |
| Saltar el guardado en Notion porque "es solo un draft" | El histórico de versiones es el valor principal del flujo. Siempre persistí. |
| Generar HTML antes de validar dependencias y fechas | Errores estructurales son caros de corregir manualmente en HTML. |
| Border-left de color en las cards de alcance | Brandbook §6 lo prohíbe explícitamente. |
| Usar hex hardcodeados en el HTML | Brandbook §1: solo CSS custom properties con tokens semánticos. |
| Preguntar módulos uno por uno | UX terrible para >5 módulos. Pedí la lista tabular completa. |
| Crear `Gantts` como database en Notion | El diseño confirmado es page-tree, no DB. Subpáginas por iniciativa, página por versión. |
| Omitir el color strip Gama Full de 12px | Es el uso principal del gradient Gama Full. Siempre presente. |

---

## Archivos del skill

- `assets/template.html` — Template HTML completo con marcadores `{{...}}`. Léelo cada vez antes de generar (puede haber sido actualizado).
- `references/brand-rules.md` — Resumen condensado del brandbook aplicado a Gantt: tokens, módulos→colores, spacing, antipatrones.
- `references/notion-flow.md` — Patrones exactos de uso de las tools `mcp__plugin_Notion_notion__*` para buscar, crear y poblar páginas.
- `references/html-generation.md` — Algoritmos: cálculo de celdas Gantt, posicionamiento de bars en `%`, mapeo módulo→color, derivación de hitos automáticos.
