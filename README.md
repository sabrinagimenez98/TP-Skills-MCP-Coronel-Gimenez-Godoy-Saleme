# TP-Skills-MCP-Coronel-Gimenez-Godoy-Saleme

Trabajo práctico de **Metodología I** — *integrar una skill propia y un MCP al proyecto odontológico*.

Este repositorio es una **copia de la fundación** del proyecto `turnos-odontologia` (Knowledge Base, roadmap `CHANGES.md`, instrucciones de agente, skills y specs de OpenSpec) sobre la que se integran las dos incorporaciones obligatorias del TP:

1. **Skill propia**: `.agents/skills/export-sin-phi/SKILL.md` (creada con la skill creadora `skill-creator` de Anthropic).
2. **MCP existente**: servidor remoto **Context7** (`https://mcp.context7.com/mcp`), configurado en `opencode.json`.

> **Estado**: Parte 1 (ficha de investigación) completa; skill integrada; MCP conectado y verificado. El change elegido — **C-14 `auditoria-exportacion`** — queda *preparado, no implementado* (el TP no pide implementarlo).

## Skill propia: `export-sin-phi`

| | |
|---|---|
| **Propósito** | Generar endpoints de exportación CSV/JSON **en servidor** + su batería de tests, cumpliendo las reglas duras R7/R8/R9/R11/R13 desde el primer commit. |
| **Cuándo usarla** | Toda tarea de exportación/descarga/portabilidad de datos (C-14 primero): "bajar un CSV", "exportar en 1 clic", "ver la auditoría en un archivo". |
| **Dónde vive** | `.agents/skills/export-sin-phi/SKILL.md` |
| **Registro** | `.agents/skill-registry.md` (tabla User Skills + compact rules + fila Change → Skills) y la tabla "Skills Disponibles" de `AGENTS.md`/`CLAUDE.md`. |
| **Cómo revisar el resultado** | `code-review` (ejes Standards/Spec) + checklist de la skill: cero PHI en archivo/logs/URLs, columnas mínimas, `tenant_id` solo del token, fila `EXPORTAR` en auditoría, ≥2 tests por comportamiento contra DB real. |

Instalación reproducible en otra máquina: copiar `.agents/skills/export-sin-phi/` y `.agents/skill-registry.md` (ambos versionados en este repo) — el agente los detecta por las referencias de `AGENTS.md`.

## Integración MCP: Context7

### Qué es y para qué se usa acá

Context7 es un servidor MCP remoto que expone **documentación oficial y versionada de librerías** a la hora de escribir código (en lugar de apoyarse en conocimiento de entrenamiento, que suele estar desactualizado). Herramientas (solo lectura, 2 en total):

- `resolve-library-id` — resuelve el ID de Context7 de una librería (p. ej. `/websites/fastapi_tiangolo`).
- `query-docs` — trae la documentación concreta de un concepto de esa librería.

**Momento de uso**: al preparar e implementar cambios que dependan de APIs de FastAPI, SQLAlchemy, Pydantic, React/Vite o pytest — en este TP, la preparación de **C-14** (`StreamingResponse`/`Response` para los exports). Ver ficha y evidencias en `docs/skills-mcp/`.

### Requisitos

- Agente **OpenCode** (V2) — el cliente que lee esta configuración.
- Conexión a internet (servidor remoto `https://mcp.context7.com/mcp`).
- **No requiere API key ni cuenta**: funciona anónimo (límites de tasa menores). Opcional: key gratis `ctx7sk_*` desde <https://context7.com/dashboard> para subir el cupo (1.000 llamadas/mes en el plan free) — esa key va **solo** en una variable de entorno, nunca en el repo (R10).

### Configuración (reproducible, sin secretos)

Archivo `opencode.json` en la raíz del proyecto (formato OpenCode V2 — `mcp.servers`):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "context7": {
        "type": "remote",
        "url": "https://mcp.context7.com/mcp"
      }
    }
  }
}
```

Si se usa API key (opcional), agregar el header con sustitución de variable de entorno — jamás el valor literal:

```json
"headers": { "CONTEXT7_API_KEY": "{env:CONTEXT7_API_KEY}" }
```

Equivalente por CLI (escribe la config del proyecto):

```sh
opencode mcp add context7 --url https://mcp.context7.com/mcp
```

> **No usamos `.mcp.json`**: es el formato de Claude Code. Según la documentación oficial de OpenCode V2 (<https://opencode.ai/v2/docs/mcp-servers>), el cliente solo lee `opencode.json` / `.opencode/opencode.json` — por eso la configuración de ejemplo sin secretos vive en `opencode.json`.

### Comprobación de conexión

```sh
opencode mcp list
```

Salida esperada (evidencia real guardada en `docs/skills-mcp/evidencias/01-mcp-list-conexion.txt`):

```text
✓ context7  connected
│      https://mcp.context7.com/mcp
```

También desde la sesión: comando `/mcps` (muestra el estado del servidor).

### Ejecución de una herramienta real

Ejemplo (evidencia completa en `docs/skills-mcp/evidencias/02-context7-herramientas-resolve-library-id.md`):

1. Solicitud: `resolve-library-id(libraryName: "Vite", query: "React TypeScript Vite PWA service worker")`
2. Resultado: `/vite-pwa/docs` (599 snippets) y `/vitejs/vite` (2087 snippets, v4–v8).
3. Aporte: resuelve el plugin PWA que usa el frontend con documentación vigente, en lugar de recordar API del modelo.

### Problemas de instalación encontrados y solución

| Problema | Causa | Solución |
|---|---|---|
| `opencode mcp add` escribió en el config **global** (`~/.config/opencode/opencode.json`) en vez del proyecto | El CLI de esta máquina eligió el destino global al no existir `opencode.json` previo | Crear/editar `opencode.json` del proyecto a mano con el shape V2 `mcp.servers.*` (el shape directo bajo `mcp` es V1) y verificar con `opencode mcp list` desde la carpeta del proyecto |
| La salida de `opencode mcp list` por PowerShell llega envuelta en `NativeCommandError` con códigos ANSI | PowerShell interpreta el stderr de la CLI como error | Redirigir `2>$null`, quitar los códigos con `-replace "\x1b\[[0-9;]*m", ""` y filtrar líneas vacías antes de guardar la evidencia |

### Límites de acceso y datos de prueba

- **Solo lectura**: Context7 solo recibe documentación de librerías; no toca la base de datos ni el código del proyecto.
- **Nunca PHI**: las queries al MCP salen a un servicio externo → no incluir DNI, teléfonos, datos clínicos, montos ni rutas de datos reales (R9/R13). Solo nombres de librería y tema.
- **Rate limit**: al agotar el cupo responde `429` con `Retry-After` — reintentar o configurar la key opcional.

## Documentos del TP

- `docs/skills-mcp/investigacion.md` — ficha de investigación (Parte 1: necesidad, skill creadora, comparación de MCPs, fuentes).
- `docs/skills-mcp/aplicacion.md` — informe de aplicación (Parte 2: evidencias, archivos modificados, revisión crítica).
- `docs/skills-mcp/evidencias/` — registros de conexión y ejecución de herramientas.

## Flujo de trabajo

```text
KB → CHANGES.md → /opsx:propose → /opsx:apply → /opsx:archive
```

Con `export-sin-phi` cargada en Backend Aux para las tareas de C-14 y Context7 disponible para resolver documentación del stack durante la preparación.
