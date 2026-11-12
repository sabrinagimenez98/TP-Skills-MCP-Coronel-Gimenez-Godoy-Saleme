# Ficha de investigación — Parte 1 (ítems 1–4)

**TP**: integrar una skill propia y un MCP al proyecto odontológico · **Fecha de consulta de fuentes**: 2026-11-12

## 1. Necesidad detectada y change elegido

**Change elegido: `C-14 auditoria-exportacion`** (FASE 5, governance CRITICO, estado `[ ]` pendiente en `CHANGES.md`).

Justificación: es el único cambio transversal pendiente cuyas dependencias están **completamente archivadas** — `C-11 caja-senas` y `C-13 obras-sociales-liquidacion` figuran cerrados en `openspec/changes/archive/`, con specs vigentes en `openspec/specs/`. Su scope dice explícitamente **"Sin migración (lee tablas existentes)"**: la tabla `auditoria` existe desde `C-02`, el patrón de exportación CSV/JSON en servidor ya se probó en `C-13` (`tests/test_liquidaciones_export.py`), y los endpoints y RBAC que consume ya están cableados. Los otros pendientes quedan descartados: `C-17` depende de `C-14` (bloqueado), y `C-16` es frontend acoplado a media docena de features aún no cerradas en UI.

**Necesidad de skill (tarea repetible)**: `C-14` pide exportación 1-clic de *cuatro* entidades distintas (agenda, pacientes, caja, liquidaciones) más una batería de tests que se repite idéntica por cada una: aislamiento por `tenant_id`, cero PHI en el archivo (R9), RBAC por rol (recepción/odontólogo sin acceso), auditoría inmutable. Hoy ese patrón vive solo en la memoria de los cambios archivados; una skill propia lo estandariza y evita reimplementarlo a mano por entidad.

**Necesidad de MCP (acceso a información)**: implementar esos endpoints exige documentación *actual* y versionada de FastAPI (`StreamingResponse`/`Response`), SQLAlchemy async y Pydantic v2 — justo lo que el agente resuelve mal por conocimiento de entrenamiento desactualizado. Un MCP de documentación en vivo cubre esa necesidad durante la preparación e implementación de `C-14`.

## 2. Skill creadora y diseño de la skill propia

**Skill creadora encontrada**: `skill-creator`, fuente oficial **https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md** (repo `anthropics/skills`, también publicada en `skills.sh` con `npx skills add https://github.com/anthropics/skills --skill skill-creator`). Es la skill "creadora de skills" de Anthropic: guía el ciclo completo — captura de intención, borrador, casos de prueba, evals comparativos (con/sin skill), iteración y optimización de la `description` para el trigger. **Compatibilidad**: su `SKILL.md` usa frontmatter YAML `name`/`description` + instrucciones en markdown, el mismo formato que ya usan las skills de `.agents/skills/` de este repo (estándar Agent Skills, `spec/` y `template/` del propio repo); está disponible en este agente. Salvedad: sus scripts de evals (`run_eval.py` etc.) invocan `claude -p` por subprocess — útiles solo como referencia, el flujo de la skill se puede seguir manualmente en OpenCode (marcado como adaptación, no verificado en este entorno).

**Diseño de la skill propia** — nombre: `export-sin-phi`, ruta `.agents/skills/export-sin-phi/SKILL.md`:

- **Propósito**: generar endpoints de exportación CSV/JSON en servidor + su batería de tests, cumpliendo R7/R8/R9/R11 desde el primer commit.
- **Cuándo se activa**: al implementar cualquier tarea de exportación/descarga de datos (`C-14` primero) o de auditoría consultable.
- **Entradas**: entidades a exportar, spec del change en `openspec/specs/`, reglas duras de `AGENTS.md`, patrón de referencia en el export ya existente de `C-13`.
- **Procedimiento**: (1) leer spec + KB (`04` §Auditoria, `05` RN-SE-01/02, `07` Flujo 7); (2) endpoint bajo RBAC con el `tenant_id` **solo** del token; (3) generación en servidor (nunca montos ni datos calculados en cliente); (4) tests — por entidad: export solo del tenant + sin PHI (DNI/teléfono/datos clínicos) + rechazo por rol; (5) auditoría de la descarga; (6) `ruff` + `mypy` + pytest contra DB real en contenedor (R11).
- **Resultado esperado**: endpoint `GET …/exportar?formato=csv|json` funcional, ≥2 tests por comportamiento verdes, cero PHI en archivos de prueba (fixtures sintéticos, R13).
- **Criterios de revisión**: `code-review` en los ejes Standards/Spec; checklist: sin PHI en export ni logs, sin cálculo en cliente, rollback de migración no aplica (sin migración), docs `README.md` sin desviarse del contrato real (patrón `tests/test_*_documentacion.py`).

**Registro**: alta en `.agents/skill-registry.md` (tabla *User Skills* + bloque *Compact Rules* + fila en *Change → Skills* para C-14). El `.atl/skill-registry.md` es shim que redirige ahí (nota del propio registry).

## 3. Comparación de dos servidores MCP y elección

| | **Context7** (Upstash) | **Playwright MCP** (Microsoft) |
|---|---|---|
| Propósito | Docs de librerías actualizadas y ejemplos de código | Automatización de browser (navegar, clic, snapshot de accesibilidad) |
| Herramientas | `resolve-library-id`, `query-docs` (2, solo lectura) | `browser_navigate/click/snapshot/type…` (decenas, escritura en la página) |
| Instalación | Remoto `https://mcp.context7.com/mcp` o local `npx -y @upstash/context7-mcp --api-key …`; plugin `@upstash/context7-opencode`; `opencode mcp add context7 --url https://mcp.context7.com/mcp` | Local `npx @playwright/mcp@latest` (Node ≥20, descarga Chromium en el primer uso) |
| Permisos / credenciales | API key gratis `ctx7sk_*` (context7.com/dashboard) o anónimo con límite bajo; OAuth en clientes que lo soportan | Sin credenciales; browser headed por defecto; guarda local de archivos (no es barrera de seguridad real) |
| Costo | Free: 1.000 llamadas/mes; Pro $10/seat | Gratis, pero coste alto de tokens (schemas + árbol de accesibilidad por acción) |
| Riesgo para el proyecto | Mínimo: solo sale el nombre de librería + la consulta (Context7 instruye a strippear datos sensibles; R9 compatible) | Alto: opera un browser sobre la app con datos reales; contexto rápido y potencial de tocar PHI |

**Elección: Context7.** Resuelve la necesidad de información que prepara el mismo change elegido (docs actuales de FastAPI/SQLAlchemy/Pydantic para los exports de `C-14`), es de solo lectura, consume dos herramientas de contexto y no requiere browser. Playwright MCP queda documentado como candidato para una fase posterior (tests E2E de `C-16`/`C-17`), no para preparar `C-14`.

## 4. Instalación, verificación y límites

**Pasos (según docs oficiales)**: 1) crear API key gratis en `context7.com/dashboard`; 2) `opencode mcp add context7 --url https://mcp.context7.com/mcp --header 'CONTEXT7_API_KEY={env:CONTEXT7_API_KEY}'` (o plugin `opencode plugin add @upstash/context7-opencode`, o `npx ctx7 setup --opencode`); 3) guardar la key en `.env`/variables de entorno — **nunca** en el repo (R10); 4) regla en `AGENTS.md`: "cuando haya que buscar documentación de librerías, usar Context7".

**Comprobar conexión y ejecutar una herramienta**: `opencode mcp list` debe mostrar `✓ context7 connected` (o `/mcps` en la sesión); luego pedir `"use context7 con /fastapi/fastapi para StreamingResponse en FastAPI"` — eso dispara `resolve-library-id` → `query-docs`. Health check alternativo: `curl https://mcp.context7.com/ping`; inspector interactivo: `npx -y @modelcontextprotocol/inspector npx @upstash/context7-mcp`.

**Qué se podrá consultar**: documentación versionada de cualquier librería pública del stack (FastAPI, SQLAlchemy, Pydantic, React/Vite, pytest, Alembic) con ejemplos de código; nada de datos del proyecto.

**Límites para datos de prueba**: las consultas al MCP van a un servicio externo → **nunca** incluir PHI, DNI, teléfonos, montos ni rutas de datos reales en los prompts a Context7 (R9/R13); solo nombres de librería y temas. La key no se commitea (R10). Límite de tasa: 429 con `Retry-After` al agotar el cupo gratuito.

## 5. Archivos de la fundación a actualizar

- **Skill**: `.agents/skills/export-sin-phi/SKILL.md` (nuevo), `.agents/skill-registry.md` (trigger + compact rules + fila C-14), `AGENTS.md` y `CLAUDE.md` (fila en la tabla "Skills Disponibles"), KB solo si surge una regla nueva (hoy no).
- **MCP**: `.mcp.json` (config del servidor pedida por el TP — *pendiente de verificación*: los docs oficiales de OpenCode especifican `opencode.json` bajo `mcp`/`mcp.servers`, no confirmé lectura de `.mcp.json`), `README.md` (sección de configuración del MCP), `AGENTS.md` (regla "usar Context7 para docs de librerías").

## 6. Fuentes (consulta: 2026-11-12)

1. Skill creator — https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md
2. Repo Anthropic Skills (README, spec/, template/) — https://github.com/anthropics/skills
3. Context7 instalación — https://context7.com/docs/installation
4. Context7 clientes + tabla de herramientas MCP — https://context7.com/docs/resources/all-clients
5. Context7 para OpenCode — https://context7.com/docs/clients/opencode
6. Context7 planes y precios — https://context7.com/docs/plans-pricing
7. Context7 prividad de datos — https://context7.com/docs/security/data-privacy
8. OpenCode MCP servers — https://opencode.ai/docs/mcp-servers
9. Playwright MCP intro/instalación/configuración — https://playwright.dev/mcp/introduction · https://playwright.dev/mcp/installation · https://playwright.dev/mcp/configuration/options
