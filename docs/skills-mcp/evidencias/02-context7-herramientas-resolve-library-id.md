# Evidencia 02 — ejecución de herramientas reales del MCP Context7

Fecha: 2026-11-12
Herramienta: `context7 resolve-library-id` (MCP remoto `https://mcp.context7.com/mcp`)
Contexto: comprobación de que el servidor expone herramientas utilizables y que devuelven documentación real del stack del proyecto.

## Solicitud 1

- Parámetros: `libraryName: "FastAPI"`, `query: "FastAPI async SQLAlchemy session dependency per request"`
- Resultado (resumen): devolvió 5 librerías coincidentes, encabezadas por `/websites/fastapi_tiangolo` (4160 snippets, reputación alta) y `/websites/fastapi_tiangolo_reference` (1512 snippets).

## Solicitud 2

- Parámetros: `libraryName: "Vite"`, `query: "React TypeScript Vite PWA service worker"`
- Resultado (resumen): devolvió `/vitejs/vite` (2087 snippets, versiones v4–v8) y `/vite-pwa/docs` (599 snippets) — este último es el plugin PWA que usa el frontend del proyecto.

## Qué aportó

- Confirmó que Context7 resuelve IDs de librerías específicas del stack elegido (FastAPI para el backend, Vite PWA para el frontend) con documentación versionada actualizable.
- El ID `/vite-pwa/docs` es el que se usará con `query-docs` durante la preparación de C-14 y de los changes de frontend (C-16/C-17), para basar las decisiones de diseño en documentación oficial vigente y no en conocimiento entrenado del modelo.
