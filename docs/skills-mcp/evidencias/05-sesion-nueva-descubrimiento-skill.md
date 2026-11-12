# Evidencia 05 — la skill se encuentra y se usa en una sesión nueva (ítem 5)

Fecha: 2026-11-12
Prueba requerida por el TP: *"En una sesión nueva, comprueben que el agente puede encontrar y utilizar la skill mediante las instrucciones y referencias del proyecto."*

## Solicitud realizada (sesión de agente SIN contexto previo)

> "Hay que implementar la exportación de la caja del mes a CSV en el backend, con tests. Antes de proponer nada, revisá si el proyecto tiene alguna skill que sirva para esto y decírsela. Buscá en las instrucciones y el registro de skills del proyecto."

No se le indicó el nombre de la skill ni su ruta.

## Resultado obtenido

- **Skill encontrada**: `export-sin-phi` en `.agents/skills/export-sin-phi/SKILL.md`.
- **Cadena de descubrimiento**: `AGENTS.md` → tabla "Skills Disponibles" → fila Backend Aux (línea 43) → nota al pie con el trigger → `.agents/skill-registry.md` (tabla User Skills + Compact Rules) → `SKILL.md` completo.
- **Uso**: resumió correctamente el procedimiento (endpoint en servidor con tenant del JWT, formato C-13, cero PHI, fila `EXPORTAR` en auditoría, ≥2 tests por comportamiento contra DB real) y citó `tests/test_liquidaciones_export.py` como patrón — todo tomado de la skill, no de conocimiento previo.
- **Discriminación**: identificó skills que NO aplican (`observe-whatsapp`, `stripe-integration`, `vercel-react-best-practices`, `multi-stage-dockerfile`) y la auto-limitación de la skill (prohíbe CSV armado en el cliente).

## Qué aportó al TP

Acredita el ítem 5: la skill no solo está escrita y registrada, sino que es **descubrible por un agente nuevo** a través de las referencias de la fundación (`AGENTS.md` + skill-registry), sin instrucciones explícitas.

## Hallazgo crítico detectado en la prueba

`.agents/skill-registry.md` declara *"Delegator use only… sub-agents do NOT read this registry"*. La sesión nueva encontró la skill igual porque **`AGENTS.md` la lista primero** y solo después bajó al registry — el orden de las referencias es lo que hace funcional el descubrimiento. Se documenta como limitación a mantener: si alguna vez se quitara la fila de `AGENTS.md`, los sub-agentes que no lean el registry perderían la skill.
