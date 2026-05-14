---
description: Crea o actualiza una carta Gantt + WBS marca Gama Mobility. Guía un flujo interactivo, persiste la iniciativa/versión en Notion bajo "Gantts" y genera el HTML final con el brandbook aplicado.
argument-hint: "[opcional] nombre de iniciativa — si existe en Notion bajo Gantts, se carga la última versión para iterar"
---

El usuario invocó `/gantt`. Iniciá ahora mismo el flujo de creación/actualización de una carta Gantt Gama Mobility.

**Argumentos del usuario:** `$ARGUMENTS`

## Cómo proceder

1. Invocá la skill `gantt-creator` (está en este mismo plugin) — ella contiene el flujo completo, las reglas de brand y la lógica de Notion.
2. Si `$ARGUMENTS` contiene un nombre de iniciativa, usalo como pista inicial para buscar en Notion bajo "Gantts" antes de hacer la primera pregunta — si encontrás match, ofrecé cargar la última versión y proponer incremento de versión; si no hay match o `$ARGUMENTS` está vacío, arrancá desde cero con las 3 preguntas obligatorias (Nombre de iniciativa, P.O, Versión).
3. No saltes preguntas obligatorias bajo ninguna circunstancia — el contrato del comando depende de capturarlas y persistirlas.
4. Al final, generá el HTML usando el template embebido en la skill (`skills/gantt-creator/assets/template.html`) y preguntá la ruta de salida antes de escribir.

Si la skill por alguna razón no se carga, lee directamente `skills/gantt-creator/SKILL.md` desde el mismo directorio del plugin y seguí esas instrucciones.
