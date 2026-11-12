# Evidencia 04 — uso del MCP Context7 durante la preparación de C-14

Fecha: 2026-11-12
MCP: Context7 (remoto `https://mcp.context7.com/mcp`) — herramientas `context7 resolve-library-id` y `context7 query-docs`, ejecutadas desde la sesión.
Contexto: decisión de diseño **D2** del `design.md` de `c-14-auditoria-exportacion` (cómo generar y transmitir el archivo de export en el endpoint, con "export grande" del Flujo 7). La ficha `investigacion.md` §1 preveía exactamente esta necesidad: docs actuales de `StreamingResponse`/`Response` de FastAPI y SQLAlchemy async en vez de conocimiento de entrenamiento.

## Solicitud 1 — `resolve-library-id` (FastAPI)

- Parámetros: `libraryName: "FastAPI"`, `query: "StreamingResponse file download attachment Content-Disposition CSV export"`, `language: "Python"`
- Resultado (resumen): devolvió 5 librerías; la encabezada `/websites/fastapi_tiangolo` (4160 snippets, reputación alta) y `/websites/fastapi_tiangolo_reference` (1512 snippets).

## Solicitud 2 — `resolve-library-id` (SQLAlchemy)

- Parámetros: `libraryName: "SQLAlchemy"`, `query: "async session streaming results large query server-side iteration"`, `language: "Python"`
- Resultado (resumen): devolvió `/websites/sqlalchemy_en_20_orm` (7872 snippets, score 79.57) como mejor candidato para ORM async, más `/sqlalchemy/sqlalchemy` (21250 snippets, versiones 1.4/2.0).

## Solicitud 3 — `query-docs` sobre FastAPI (la que aportó al design)

- Parámetros: `libraryId: "/websites/fastapi_tiangolo"`, `query: "StreamingResponse return file download Content-Disposition attachment filename headers"`, `language: "Python"`
- Resultado (resumen):
  - `StreamingResponse(content, status_code, headers, media_type, background)`: acepta **async iterable o iterable regular** como `content` y lo transmite como cuerpo de la respuesta ("useful for large responses"), con `charset` por defecto `utf-8`.
  - `FileResponse(path, …, filename=…, content_disposition_type="attachment")`: si `filename` está seteado se incluye en el `Content-Disposition`; agrega `Content-Length`/`Last-Modified`/`ETag` — pero **requiere un archivo ya existente en disco** (`path`).

## Solicitud 4 — `query-docs` sobre SQLAlchemy async (la que aportó al design)

- Parámetros: `libraryId: "/websites/sqlalchemy_en_20_orm"`, `query: "async session stream results chunked iteration yield_per large result set without loading all rows"`, `language: "Python"`
- Resultado (resumen): `AsyncResult.yield_per(n)` configura fetch por chunks; `yield_per` equivale a `stream_results` (**server-side cursors** donde el backend lo soporta) y evita que el ORM materialice todas las filas en memoria ("buffer only a limited number of rows… optimized for iteration through very large result sets (e.g. > 10K rows)"); combinable con `Result.partitions()`.

## Qué aportó al design (decisión concreta)

- **D2 (`StreamingResponse` con generador + `yield_per`, no `FileResponse` ni `Response` en memoria)** se tomó sobre documentación vigente en lugar de memoria del modelo:
  - Contra `Response` completo: `StreamingResponse` está pensado explícitamente para "large responses" y acepta un generador async → cumple el caso "export grande" del Flujo 7 sin bufferizar el archivo.
  - Contra `FileResponse`: la doc confirma que necesita `path` a un archivo en disco → habría que persistir y limpiar temporales para CSVs on-the-fly; descartada.
  - El header `Content-Disposition: attachment; filename=…` se fija en `headers`/`media_type` de la respuesta (el `content_disposition_type="attachment"` de `FileResponse` confirma la convención `attachment` usada en C-13).
  - `yield_per` + server-side cursors resuelve el otro lado de la moneda: que la query de SQLAlchemy async no cargue todas las filas en RAM mientras el generador las va emitiendo (tarea 1.1 de `tasks.md`).
- **Registro**: solicitud + resultado quedan aquí y la decisión D2 los cita como fuente (`design.md` → evidencia `04`).
- **Límite R9 respetado**: las consultas contuvieron solo nombres de librería y temas (`FastAPI`, `SQLAlchemy`, `StreamingResponse`…); ningún dato del proyecto ni de pacientes.
