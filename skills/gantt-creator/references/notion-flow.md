# Notion flow — Gantts en Notion

> Patrones específicos para usar el MCP `mcp__plugin_Notion_notion__*` desde esta skill. Si las tools cambian de nombre en el plugin, el principio sigue siendo el mismo: page-tree con `Gantts` como raíz.

## Estructura definitiva del árbol

```
Gantts                              ← página raíz (workspace-level)
├── Portal de Siniestros            ← subpágina por iniciativa
│   ├── v0.1                        ← una subpágina por versión
│   ├── v0.2
│   └── v0.3
├── Migración ERP Gama
│   ├── v0.1
│   └── v0.2
└── ...
```

Una iniciativa nueva crea su subpágina. Una nueva versión crea otra subpágina debajo. Las versiones **nunca** se sobreescriben — la idea es histórico auditable.

## Tools relevantes

Con el plugin Notion oficial, las tools llegan namespaceadas como `mcp__plugin_Notion_notion__notion-<verb>`:

| Tool                         | Uso aquí                                                  |
|------------------------------|-----------------------------------------------------------|
| `notion-search`              | Encontrar `Gantts`, encontrar subpágina de iniciativa     |
| `notion-fetch`               | Leer contenido de la última versión para iterar           |
| `notion-create-pages`        | Crear `Gantts`, crear iniciativa, crear versión           |
| `notion-update-page`         | (Raro) actualizar metadata de una versión ya creada       |

## Paso a paso

### 1. Encontrar o crear `Gantts`

```
notion-search:
  query: "Gantts"
  query_type: "internal"
```

Filtros mentales sobre la respuesta:
- Buscá item con `title == "Gantts"` exacto, tipo `page`, sin parent (workspace root) o con parent que sea un team space conocido.
- Si hay múltiples candidatos, preferí el más recientemente editado y preguntá al usuario para confirmar.
- Si no hay match, creala:

```
notion-create-pages:
  pages: [{
    title: "Gantts",
    content: "Repositorio de cartas Gantt + WBS de Gama Mobility. Cada subpágina es una iniciativa; cada subpágina de iniciativa es una versión."
  }]
```

Guardá el `page_id` y `url` resultantes en memoria de la conversación.

### 2. Encontrar o crear la subpágina de iniciativa

```
notion-search:
  query: "<nombre iniciativa>"
  query_type: "internal"
```

Sobre la respuesta:
- Filtrá por items cuyo parent sea el `page_id` de `Gantts` (resuelto en paso 1).
- Match exacto → usalo.
- Match fuzzy (espacios, acentos, mayúsculas) → preguntá al usuario si es el mismo proyecto:
  > "Encontré `Portal Siniestros` bajo Gantts (creado 2026-04-12). ¿Es la misma iniciativa que estás versionando?"
- Sin match → creala:

```
notion-create-pages:
  parent: { page_id: "<id-de-Gantts>" }
  pages: [{
    title: "<nombre iniciativa>",
    content: "Iniciativa Gama Mobility. P.O: <po>. Creada con /gantt — cada subpágina es una versión del plan."
  }]
```

### 3. Listar versiones existentes (si la iniciativa ya existía)

```
notion-fetch:
  urls: ["<url-iniciativa>"]
```

Parseá las children pages buscando títulos que matchen el patrón `v\d+\.\d+`. Ordenalas por versión semántica. La "última versión" es la mayor.

Proponé al usuario:
- **Incrementar minor** (por defecto): si la última fue `v0.3`, proponer `v0.4`.
- **Override**: el usuario puede dictar otra (`v1.0` para marcar release, `v0.3.1` para hotfix). Aceptá.

### 4. (Si el usuario eligió iterar) leer la versión anterior

```
notion-fetch:
  urls: ["<url-version-anterior>"]
```

Extraé las tablas WBS / Hitos / Riesgos del contenido. Mostrale al usuario un resumen tipo:

```
Tu última versión (v0.2) tiene:
- 6 módulos (M0, M1, M2, M3, M4, PM)
- 32 paquetes de trabajo
- 6 hitos firmables
- Inicio 2026-06-01, fin 2026-08-24 (12 semanas)

¿Qué querés ajustar para v0.3? (agregar/eliminar módulos, mover fechas, agregar hito, etc.)
```

Capturá los deltas y mergea con la base. La idea es no obligar al usuario a re-pegar toda la tabla si solo cambia 2 fechas.

### 5. Crear la subpágina de la nueva versión (al final del flujo, después de generar HTML)

```
notion-create-pages:
  parent: { page_id: "<id-iniciativa>" }
  pages: [{
    title: "v<X.Y>",
    properties: {
      // si el plugin permite custom properties, agregar: created_by, po, fecha
    },
    content: <bloques estructurados, ver abajo>
  }]
```

### 6. Estructura del contenido de la página de versión

Usá bloques Notion nativos (no embebas el HTML completo). Estructura recomendada:

```markdown
# Metadata
- Iniciativa: <nombre>
- Product Owner: <po>
- Versión: v<X.Y>
- Fecha de creación: <YYYY-MM-DD>
- Inicio proyecto: <YYYY-MM-DD>
- Fin proyecto: <YYYY-MM-DD>
- Total semanas: <N>
- Unidad de tiempo: <HH/Semanas/Ambas>

# Alcance (Módulos)
<para cada módulo: pill + nombre + descripción corta si la tiene>

# WBS — Estructura de Desglose del Trabajo
<tabla con columnas: WBS · Paquete · HH · Sem · Owner · Predecesor · Inicio · Fin · Sprint>

# Hitos firmables
<tabla: # · Hito · Semana · Cómo se valida>

# Ruta crítica
<si el usuario la definió, un quote/paragraph con la cadena A→B→C>

# Riesgos prioritarios
<tabla: Riesgo · Mitigación · si la lista existe>

# Output local
```
Ruta absoluta del HTML generado: <path>
Fecha de generación: <timestamp>
```
```

Los bloques `table` en la API de Notion necesitan estructura específica (`table_width`, `has_column_header`, `children` con `table_row` blocks). Si la tool soporta markdown directo, usalo; si requiere blocks JSON, construilos.

### 7. Reportar la URL al usuario

Después de crear la versión, mostrale la URL para que pueda compartirla. Si la creación falla, no rompas el flujo del HTML — el HTML local se escribe igual; reportá el error de Notion y ofrece reintentar.

## Edge cases

- **Sin permisos en Notion**: si la búsqueda devuelve `unauthorized` o las tools no están disponibles, advertí al usuario:
  > "No pude acceder a Notion para guardar la versión. ¿Querés que genere igualmente el HTML local y volvemos a Notion después?"
  Si dice que sí, continuá con la generación HTML y omití los pasos de creación.

- **Iniciativa con nombre ambiguo** (existe `Portal` y `Portal de Siniestros`): listale los matches y dejá que elija. Nunca asumas.

- **Versión ya existe** (intentaste crear `v0.3` y ya hay `v0.3` por una corrida fallida previa): proponé `v0.3.1` o sobreescribir con confirmación explícita.

- **Renombre de iniciativa**: si el usuario quiere renombrar, pedile que lo haga en Notion primero y vuelva a invocar — no manejes renombres desde la skill (riesgo de duplicar).
