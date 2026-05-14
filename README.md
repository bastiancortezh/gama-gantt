# gama-gantt — Plugin Claude Code · Gama Mobility

Plugin para generar **cartas Gantt + WBS marca Gama Mobility** desde una conversación corta con Claude. Persiste la metadata e iteraciones en Notion bajo la página `Gantts`, y exporta un HTML standalone con el brandbook aplicado (light + dark mode, color strip Gama Full, sin `border-left` coloreado en cards, etc.).

Invocable como **`/gantt`** desde Claude Code o Claude Cowork.

---

## Qué hace

1. Pregunta lo obligatorio del inicio: **Nombre de iniciativa · P.O · Versión**.
2. Busca/crea la página padre `Gantts` en Notion y la subpágina de la iniciativa.
3. Si la iniciativa ya existía, detecta la última versión y propone incremento (`v0.1 → v0.2 → v0.3`).
4. Captura: cantidad de sprints/hitos, unidad de tiempo (HH/Semanas/Ambas), fechas, módulos con owner/predecesor/dependencia.
5. Crea una **subpágina por versión** en Notion con la data estructurada (histórico auditable).
6. Genera el HTML standalone con el brandbook Gama aplicado.
7. Pregunta al final dónde guardar el HTML local.

---

## Estructura

```
gama-gantt-plugin/
├── .claude-plugin/
│   └── plugin.json                  # manifest del plugin
├── commands/
│   └── gantt.md                     # entry point del slash command /gantt
├── skills/
│   └── gantt-creator/
│       ├── SKILL.md                 # flujo completo + contrato
│       ├── references/
│       │   ├── brand-rules.md       # tokens, colores, antipatterns
│       │   ├── notion-flow.md       # patrones MCP Notion
│       │   └── html-generation.md   # algoritmos de generación
│       └── assets/
│           └── template.html        # template HTML con {{placeholders}}
└── README.md
```

---

## Instalación

### Opción A · Local (development)

Para iterar el plugin localmente sin publicar:

```bash
# Desde Claude Code en el repo del plugin:
/plugin install ./
# o desde otro directorio, apuntando al path absoluto:
/plugin install "C:/Users/BastianCortezHernand/OneDrive - GAMA LEASING OPERATIVO SPA/Projects/DEV/gama-gantt-plugin"
```

Cualquier edición a los archivos del plugin se refleja en la próxima invocación — no requiere reinstalar.

### Opción B · Marketplace privado (Gama)

1. `git init` en la raíz del plugin, primer commit.
2. Push a un repo privado de Gama (GitHub/Azure DevOps).
3. Agregar al `marketplace.json` del workspace (ver doc oficial de plugins Claude Code).
4. `/plugin install gama-gantt` desde cualquier máquina con acceso al marketplace.

### Opción C · Claude Cowork

Mismo flujo que Claude Code:

1. Sube el plugin al repo accesible desde Cowork.
2. En la sesión de Cowork: `/plugin install <url-del-repo>`.
3. `/gantt` queda disponible en todas las sesiones futuras del workspace.

> **Nota Cowork:** las tools MCP de Notion deben estar habilitadas en la configuración del workspace. Si no hay acceso a Notion, la skill genera igualmente el HTML local y reporta el error de persistencia (no rompe el flujo).

---

## Requisitos

- **Claude Code** o **Claude Cowork** con soporte de plugins.
- **MCP Notion habilitado** (`mcp__plugin_Notion_notion__*`). Sin esto, la persistencia falla pero el HTML se genera igual.
- **Acceso de escritura** en el workspace Notion donde vive (o se va a crear) la página `Gantts`.
- Para el logo en el HTML: el path por defecto es relativo a `proyecto-dual-write/brandbook/RGB/png/`. Si el HTML va a un directorio donde esa relativa no funciona, la skill ofrece alternativas (absoluta o base64).

---

## Cómo usar — ejemplo

### Primera invocación de una iniciativa nueva

```
> /gantt

Antes de armar el Gantt necesito 3 datos obligatorios:
1. Nombre de la iniciativa
2. Product Owner
3. Versión (ej: v0.1 si es nueva)

> Portal de Siniestros, María Pérez (Operaciones), v0.1

✓ Creé "Gantts" en Notion
✓ Creé subpágina "Portal de Siniestros" bajo Gantts
✓ Versión confirmada: v0.1

¿Cuántos Sprints/hitos tendrá? [2|3|4|6|8|otro]
> 3

¿Unidad de tiempo? [HH|Semanas|Ambas]
> Ambas

Fecha de inicio del proyecto:
> 2026-06-01

Fecha de término:
> 2026-08-24

[Ok, 12 semanas, 3 meses. Listo para los módulos.]

Pasame los módulos en formato CSV-like... [continúa]
```

### Re-invocación (iterar versión)

```
> /gantt Portal de Siniestros

✓ Encontré "Portal de Siniestros" en Notion (3 versiones: v0.1, v0.2, v0.3)
✓ Propongo v0.4

¿Iterar desde v0.3 (cargo la data y capturo deltas) o partir de cero?
> iterar

Tu v0.3 tiene: 6 módulos, 32 paquetes, 12 semanas (2026-06-01 → 2026-08-24).
¿Qué cambia para v0.4?
> mover go-live a S13, agregar paquete M4.7 hardening adicional

[Ok, aplico los deltas y genero la nueva versión.]
```

---

## Reglas no-negociables del brandbook aplicadas

- Cards **sin** `border-left` coloreado — categoría va en el pill dentro del título.
- Color strip Gama Full (12px, los 4 colores) fijo al fondo de pantalla, siempre presente.
- Tokens CSS — nunca hex hardcodeados.
- Logo Gama Mobility con swap automático claro/oscuro.
- Spacing escala 4/8/16/24/32/48/64 — sin valores arbitrarios.
- Tipografía Mont (headings) + Montserrat (body) desde Google Fonts.
- Toggle de tema en `.header-right` — no botón flotante.

Fuente de verdad completa: `proyecto-dual-write/brandbook/brandbook-agent.md` en el repo Gama.

---

## Versionado del plugin

- `v0.1.0` — primer release: flujo end-to-end, Notion persistence, HTML con dark mode.

Para nuevas features, bump el `version` en `.claude-plugin/plugin.json` y tag en git.

---

## Troubleshooting

| Síntoma | Causa probable | Fix |
|---|---|---|
| `/gantt` no aparece | Plugin no instalado o no recargado | `/plugin list` → si no está, reinstalar. Si está, reiniciar la sesión. |
| Falla al crear página en Notion | MCP Notion sin permisos | Reautenticar el MCP. La skill seguirá generando el HTML local. |
| Logo aparece roto en el HTML | Path relativo no resuelve | Re-generar el HTML eligiendo "absoluta" o "base64" cuando la skill pregunte. |
| Las bars del Gantt salen torcidas | Fechas fuera del rango del proyecto | Revisar que `inicio` y `fin` de cada paquete caigan dentro de `fechaInicio`/`fechaFin` del proyecto. |
| Cards con border-left coloreado | El usuario editó el template y reintrodujo `border-left` | Revisar `skills/gantt-creator/assets/template.html`. Brandbook lo prohíbe. |

---

## Autoría

Plugin desarrollado para Gama Mobility · DEV team. Brandbook por Vicente Araos (2026).
Documento confidencial de Gama Mobility — prohibida su reproducción sin autorización.
