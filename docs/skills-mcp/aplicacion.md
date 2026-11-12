# Informe de aplicación — Parte 2 (ítems 5–8)

**TP**: integrar una skill propia y un MCP al proyecto odontológico · **Fecha**: 2026-11-12
Continuación de la ficha [investigacion.md](investigacion.md) (Parte 1). Evidencias en [evidencias/](evidencias/).
Historial de esta parte: `2047078` (config MCP + evidencia 01) → `9d7940e` (skill + registro + docs) → `9091ef2` (change C-14 + evidencias 03-04).

## 1. Qué se hizo

Se completaron los cuatro frentes del TP.

**Skill propia.** Se creó `export-sin-phi` (`.agents/skills/export-sin-phi/SKILL.md`, 94 líneas) siguiendo el ciclo de `skill-creator` de Anthropic: captura de intención, borrador a partir de la necesidad de C-14, casos de prueba y optimización de la `description` para el trigger.
La adaptación declarada: los scripts de evals de `skill-creator` invocan `claude -p` por subprocess y no son ejecutables en OpenCode, así que la iteración de casos y la `description` se resolvieron manualmente siguiendo su guía.
La skill quedó registrada en `.agents/skill-registry.md` (tabla User Skills, compact rules y fila C-14) y referenciada en `AGENTS.md`/`CLAUDE.md` (fila Backend Aux + sección de Integración MCP).

**MCP Context7.** Quedó conectado y operativo sin API key: config anónima en `opencode.json` (shape V2 `mcp.servers.context7` → `https://mcp.context7.com/mcp`, sin secretos).
La conexión se comprobó con `opencode mcp list` (evidencia 01) y el servidor se usó de verdad en la preparación: `resolve-library-id` (evidencia 02) y `query-docs` (evidencia 04) corrieron sobre FastAPI y SQLAlchemy para fundar la decisión de diseño D2.

**Change preparado.** `openspec/changes/c-14-auditoria-exportacion/` quedó con 4/4 artefactos de planning (proposal, design, specs, tasks).
`openspec validate --strict` dio **OK** (re-ejecutado al cerrar este informe: *"Change 'c-14-auditoria-exportacion' is valid"*).
Quedaron **17 escenarios** (happy path, errores de negocio y casos de borde) distribuidos en 9 requirements, y **16 tareas sin marcar**, cada una con verificación por escenario (`→ Escenario: "<nombre>"`).

**Roadmap.** `CHANGES.md` §[C-14] pasó a `[~] preparado, no implementado`, con la salvedad explícita de "cero tareas verificadas: pendiente de implementación vía `/opsx:apply`".

## 2. Evidencias

Cinco registros, uno por ítem verificado; los cinco son archivos guardados en el repo, no transcripciones:

- [evidencias/01-mcp-list-conexion.txt](evidencias/01-mcp-list-conexion.txt) — `opencode mcp list` desde la carpeta del TP muestra `✓ context7 connected`: prueba de conexión real, sin key.
- [evidencias/02-context7-herramientas-resolve-library-id.md](evidencias/02-context7-herramientas-resolve-library-id.md) — `resolve-library-id` resuelve IDs del stack real (`/websites/fastapi_tiangolo`, `/vite-pwa/docs`): el servidor expone herramientas utilizables que devuelven docs del proyecto.
- [evidencias/03-uso-skill-export-sin-phi.md](evidencias/03-uso-skill-export-sin-phi.md) — aplicación de la skill en la preparación de C-14, con tabla antes→después por artefacto (17 escenarios, mapeo tarea↔escenario, ≥2 casos por comportamiento).
- [evidencias/04-uso-context7-en-preparacion.md](evidencias/04-uso-context7-en-preparacion.md) — las consultas a Context7 que fundaron la decisión **D2** del design (`StreamingResponse` + `yield_per`, y por qué no `FileResponse`), tomadas sobre documentación vigente.
- [evidencias/05-sesion-nueva-descubrimiento-skill.md](evidencias/05-sesion-nueva-descubrimiento-skill.md) — sesión nueva sin contexto: encontró y usó la skill solo por las referencias del proyecto (ítem 5), e incluye el hallazgo crítico de la §4.d.

## 3. Archivos modificados y motivo

Fuente: `git log --stat` de los cuatro commits de la parte (fundación `18e22ba` incluida como base; los archivos TP son de `2047078`, `9d7940e` y `9091ef2`).

| Archivo | Motivo |
|---|---|
| `.agents/skills/export-sin-phi/SKILL.md` (nuevo, 94 líneas) | Skill propia: propósito, triggers, reglas de dominio (RN-SE-01/02, R4/R7/R8/R9/R11/R13), procedimiento en 7 pasos y checklist de revisión |
| `.agents/skill-registry.md` (+11 líneas) | Alta de la skill: fila en User Skills, bloque de compact rules y fila en Change → Skills para C-14 |
| `AGENTS.md` (±12 líneas) | Fila `export-sin-phi` en la tabla "Skills Disponibles" (Backend Aux) con su nota de trigger + nueva sección "Integración MCP (Context7)" con propósito, cuándo usarlo y límites |
| `CLAUDE.md` (±12 líneas) | Mismas dos secciones, en espejo con `AGENTS.md` (la fundación los mantiene sincronizados) |
| `README.md` (nuevo, 121 líneas) | Sección "Integración MCP: Context7": qué es, requisitos (sin key), configuración reproducible sin secretos, comprobación, ejecución de una herramienta, troubleshooting y límites de datos |
| `opencode.json` (nuevo, 11 líneas) | Config reproducible del MCP en el shape V2 `mcp.servers.*`, anónima (sin headers ni secretos) |
| `CHANGES.md` (1 línea cambiada) | Estado de `[C-14]`: de `[ ]` pendiente a `[~] preparado, no implementado` + puntero a artefactos y evidencias |
| `openspec/changes/c-14-auditoria-exportacion/` (5 archivos: `.openspec.yaml`, `proposal.md`, `design.md`, `specs/auditoria-exportacion/spec.md`, `tasks.md`) | Change preparado con el flujo SDD: 4/4 artefactos, 17 escenarios en 9 requirements, 16 tareas |
| `docs/skills-mcp/investigacion.md` (nuevo, 68 líneas) | Ficha Parte 1 (ítems 1–4): necesidad y change elegido, skill creadora, comparación de dos MCPs, límites y fuentes |
| `docs/skills-mcp/evidencias/01..05` (nuevos) | Registros de conexión, ejecución de herramientas, uso de la skill, uso en el design y test de sesión nueva |
| `docs/skills-mcp/aplicacion.md` (nuevo) | Este informe (Parte 2, ítem 8) |
| `ENTREGA.md` (nuevo, raíz) | Índice de la entrega con campos a completar (integrantes, URL, commit, video) |

Alcance de la modificación: la parte 2 del TP **no tocó** `src/`, `tests/`, los specs vigentes de `openspec/specs/` ni los changes archivados — solo se agregó documentación, configuración y planning.

## 4. Revisión crítica

Se contrastaron las decisiones de la ficha y las pruebas contra los archivos **reales** del repo (leídos, no citados de memoria).
Cuatro hallazgos confirmados, con su antes y su después:

**a. La API key que la ficha daba como paso 1 de instalación no hace falta (hallazgo principal).**
- **ANTES** (ficha `investigacion.md` §4): *"Pasos (según docs oficiales): 1) crear API key gratis en `context7.com/dashboard`; 2) `opencode mcp add … --header 'CONTEXT7_API_KEY=…'`"* — la key aparecía como requisito de instalación.
- **DESPUÉS** (`README.md` §Requisitos): *"No requiere API key ni cuenta: funciona anónimo (límites de tasa menores)"*; `opencode.json` quedó sin ningún `headers`, y `opencode mcp list` conectó igual (evidencia 01).
- **Fuente que permitió corregirlo**: README oficial de Context7 ("No API key is needed") + verificación empírica en este entorno.
- La key opcional `ctx7sk_*` quedó documentada solo como forma de subir el cupo (1.000 llamadas/mes), y siempre en variable de entorno, nunca en el repo (R10).

**b. No se creó el `.mcp.json` que la ficha preveía.**
- **ANTES** (ficha §5): preveía crear `.mcp.json` (formato Claude Code) como "config del servidor pedida por el TP", marcándola ella misma como *pendiente de verificación*.
- **DESPUÉS**: verificado en la documentación oficial de OpenCode V2 que el cliente solo lee `opencode.json` / `.opencode/opencode.json`; `Test-Path .mcp.json` → `False` (no existe: no se creó un archivo muerto).
- El README explica la decisión con la cita a la doc oficial en vez de duplicar la config.

**c. Problema real de instalación: el CLI escribió al config global en vez del proyecto.**
- **ANTES** (ficha §4, paso 2): `opencode mcp add context7 --url …` presentado como el camino estándar.
- **DESPUÉS**: en esta máquina el comando escribió en `~/.config/opencode/opencode.json` (global) porque no existía `opencode.json` previo en el proyecto → se creó el `opencode.json` del proyecto a mano con el shape V2 `mcp.servers.*` (el shape directo bajo `mcp` es V1).
- Quedó en la tabla "Problemas de instalación encontrados y solución" del README, con la verificación `opencode mcp list` desde la carpeta del proyecto.

**d. El descubrimiento de la skill depende del orden de las referencias.**
- **ANTES** (supuesto implícito al registrarla): darla de alta en `.agents/skill-registry.md` bastaría para que cualquier agente la encuentre.
- **DESPUÉS** (evidencia 05 + `.agents/skill-registry.md`, línea 3): el registry declara *"Delegator use only… sub-agents do NOT read this registry"*.
- La sesión nueva encontró la skill igual porque **`AGENTS.md` la lista antes** (tabla "Skills Disponibles" → nota de trigger → recién ahí el registry → `SKILL.md` completo). Quedó documentado como limitación a mantener: quitar la fila de `AGENTS.md` rompería el descubrimiento para los sub-agentes que no lean el registry.

**Verificación ejecutada al redactar este informe**: `openspec validate c-14-auditoria-exportacion --strict` → *valid*; conteo directo sobre los archivos: 17 `#### Scenario`, 9 `### Requirement`, 16 `- [ ]` en `tasks.md`; `Test-Path .mcp.json` → `False`; `git log --stat` como fuente de la tabla de la §3.

## 5. Limitaciones

Lo que quedó fuera del alcance o pendiente, declarado para que no se lea como omisión:

- **Evals de `skill-creator` no ejecutados**: sus scripts requieren `claude -p`; la `description` y los casos se adaptaron manualmente (adaptación declarada, no un eval comparativo con/sin skill).
- **C-14 preparado pero NO implementado ni archivado**: es el alcance del TP; `CHANGES.md` marca `[~]` con cero tareas verificadas y las specs vigentes quedaron sin tocar.
- **Context7 en modo anónimo**: cupo de tasa bajo; un `429` con `Retry-After` es posible ante uso intenso (la key opcional queda pendiente de decisión).
- **Repo del TP sin remoto configurado todavía**: el push a GitHub queda pendiente (campo de `ENTREGA.md`).
- **Video de demostración pendiente**: hasta 5 min, enlace a completar en `ENTREGA.md`.
