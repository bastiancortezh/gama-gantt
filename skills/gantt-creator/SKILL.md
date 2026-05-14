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

Este es el paso largo y el más sensible a la UX. **Ofrecé siempre las dos vías al usuario antes de empezar**:

```
Hay dos formas de capturar los módulos. ¿Cuál prefieres?

A) Guiado paso a paso (default · recomendado para definir desde cero)
   Vamos sprint por sprint. Para cada sprint te pregunto sus módulos y para
   cada módulo: nombre, HH/Semanas, owner, predecesor, fechas. Al cerrar
   cada sprint te muestro el resumen para confirmar antes de pasar al siguiente.

B) Pegar estructura completa (más rápido si ya tenés la tabla armada)
   Pegás todos los módulos de una en formato CSV/Markdown, o subís un .csv/.xlsx
   siguiendo el template (te paso el formato si lo necesitas).
```

Si el usuario no elige explícitamente, asumí **A (guiado)**. Si el contexto sugiere que ya tiene la data (mencionó un Excel, una planilla, un Jira export, etc.), proponé **B** activamente.

#### 4.A — Flujo guiado (iterativo por sprint)

Para cada sprint declarado en el Paso 3, repetí este micro-loop:

1. **Anclar el sprint**. Preguntá:
   ```
   ### Sprint <N> de <total>
   ¿Cómo se llama este sprint? (ej: "Sprint 1", "Discovery", "Hito Go-Live")
   ¿Fecha de inicio y fecha de término del sprint? (caen dentro del rango del proyecto)
   ¿Cuántos módulos/paquetes vamos a meter en este sprint?
   ```
   Validá que las fechas caigan dentro de `[fechaInicio, fechaFin]` global.

2. **Capturar cada módulo del sprint** (loop interno):
   ```
   #### Módulo <i>/<M> del sprint <N>
   - CODIGO (ej: M1.1, M0.3) — único en todo el proyecto
   - Nombre corto (lo que va en la celda "Paquete")
   - <si la unidad es HH o Ambas> HH estimadas
   - <si la unidad es Semanas o Ambas> Semanas (ej: S2 o S3-S4)
   - Owner (rol o persona)
   - Predecesor (CODIGO o "—")
   - Fecha de inicio (default: inicio del sprint)
   - Fecha de fin (default: fin del sprint)
   - Color/Categoría del módulo padre (M0/M1/M2/M3/M4/PM/QA — ver brand-rules.md)
   ```
   Para campos con default razonable, ofrecelo entre paréntesis y aceptá un "ok" del usuario sin re-tipearlos.

3. **Cerrar el sprint** mostrando un resumen breve y pedí confirmación:
   ```
   ### Resumen Sprint <N> · "<nombre>"
   - 3 módulos: M1.1 (Análisis · 16 HH), M1.2 (Backend · 40 HH), M1.3 (Frontend · 32 HH)
   - Total: 88 HH · S1-S3 (2026-06-01 → 2026-06-21)
   - Owners: Funcional (1), Backend (1), Frontend (1)
   - ¿Confirmás y pasamos al Sprint <N+1>, o ajustamos algo?
   ```
   Si dice ajustar, identificá qué módulo y aplicá el cambio sin re-preguntar todo.

4. **Cuando se cerraron los N sprints**, mostrá el resumen global:
   ```
   ### Proyecto completo
   - <count_modulos> módulos en <count_sprints> sprints
   - <total_hh> HH totales · <N_weeks> semanas (<fechaInicio> → <fechaFin>)
   - Distribución por sprint: Sprint 1 (88 HH) · Sprint 2 (120 HH) · Sprint 3 (60 HH)
   - ¿Avanzamos a hitos firmables (paso 5) o ajustamos algo?
   ```

> **Importante:** mantené el estado de los sprints capturados en la memoria de la conversación. Si el usuario quiere "volver atrás" a corregir el Sprint 1 cuando ya está en el Sprint 3, usa la última versión correcta del Sprint 1, no re-preguntes desde cero.

#### 4.B — Flujo bulk (paste o archivo)

Ofrecé el formato y el template:

```
Pegame la tabla completa en cualquiera de estos formatos. Una fila por módulo,
delimitador `|`, `\t` o `,`. Encabezado obligatorio (orden libre, los detecto):

CODIGO | NOMBRE | HH | SEMANAS | OWNER | PREDECESOR | FECHA_INICIO | FECHA_FIN | SPRINT | COLOR

Ejemplo:
M1.1 | Análisis funcional       | 16  | S2    | Funcional | —     | 2026-06-01 | 2026-06-07 | Sprint 1 | green
M1.2 | Backend CRUD             | 40  | S3-S4 | Backend   | M1.1  | 2026-06-08 | 2026-06-21 | Sprint 1 | green
M2.1 | Frontend portal cliente  | 56  | S3-S5 | Frontend  | M1.1  | 2026-06-08 | 2026-06-28 | Sprint 2 | orange

Tip: para pegar desde Excel, copia el rango con encabezado — el delimitador `\t` se detecta solo.
Si preferís subir un archivo, podés pasarme un .csv o .xlsx siguiendo este mismo orden de columnas
(podés bajar el template desde `assets/import-template.csv` del plugin).

Reglas:
- HH y SEMANAS son opcionales según la unidad declarada en el paso 3:
  HH → solo HH · Semanas → solo SEMANAS · Ambas → las dos
- PREDECESOR vacío o "—" significa sin dependencia
- COLOR es opcional: si no lo especificás, te propongo el mapeo automático según orden de aparición
  (ver brand-rules.md). Valores: purple | green | orange | red | neutral | pm | qa
- SPRINT debe coincidir con uno de los nombres que vamos a usar (puedo proponerlos si querés)
```

Si el usuario sube un archivo Excel/CSV, **leelo** y mostralo parseado como tabla markdown para que confirme antes de continuar. Mostrá explícitamente:
- Cantidad de filas leídas
- Sprints detectados
- Módulos con campos faltantes (si los hay, pedí completarlos sin re-leer el archivo)

Después de parsear, mostrá un resumen idéntico al del paso 4.A.4 (proyecto completo) y pedí confirmación.

#### Validaciones críticas (aplican a ambos flujos)

Antes de avanzar al Paso 5:

- [ ] Cada CODIGO es único en el proyecto.
- [ ] Cada PREDECESOR referencia un CODIGO que existe en la lista (o es `—`/vacío).
- [ ] No hay ciclos en el grafo de predecesores.
- [ ] Cada fecha de módulo cae dentro de `[fechaInicio_proyecto, fechaFin_proyecto]`.
- [ ] La `FECHA_INICIO` del módulo es ≤ `FECHA_FIN`.
- [ ] Cada SPRINT declarado tiene ≥ 1 módulo.
- [ ] La cantidad de sprints capturados == cantidad declarada en Paso 3 (si difiere, preguntá si bajamos/subimos el count o si hay un error).

Si algo falla, no avances al Paso 5. Mostrá los errores agrupados (no uno por uno) y pedí corregir.

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
