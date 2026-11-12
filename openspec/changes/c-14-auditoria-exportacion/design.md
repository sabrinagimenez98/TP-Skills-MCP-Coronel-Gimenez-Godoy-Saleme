# Design: `c-14-auditoria-exportacion`

## Context
La tabla `auditoria` existe desde `C-02` (append-only por `GRANT/REVOKE`, índice `(tenant_id, entidad, entidad_id, created_at)`) y cada change desde `C-05` ya inserta eventos; falta el **lectura** (endpoint con filtros) y la **exportación** de las cuatro entidades. El patrón de export en servidor ya se probó una vez en `C-13` (`tests/test_liquidaciones_export.py`, decisiones D7/D8): CSV UTF-8+BOM/`,`/RFC 4180, JSON con `Decimal` como string, `Content-Disposition` y `Accion.EXPORTAR` en la matriz. Este change estandariza ese patrón a las 4 entidades sin reimplementarlo por entidad. Ver `proposal.md` para el motivo.

Preparación con la skill propia del proyecto (`.agents/skills/export-sin-phi/SKILL.md`): su "Procedimiento" y su batería de tests por entidad moldean los escenarios de la spec y el mapeo tarea→escenario; evidencia en `docs/skills-mcp/evidencias/03-uso-skill-export-sin-phi.md`.

## Goals / Non-Goals

**Goals:**
- `GET /api/auditoria` filtrable (entidad/período/actor) y append-only de punta a punta.
- Export CSV/JSON idéntico en contrato para las 4 entidades, con RBAC `EXPORTAR`, aislamiento por tenant y cero PHI fuera de lo declarado.
- Cada descarga audita su propia fila `EXPORTAR`.
- Volumen grande degrada a job asíncrono sin cambiar el contrato de error.

**Non-Goals:**
- UI de auditoría/export en la PWA interna (eso es `C-17`); acá sólo la página legal pública.
- Presentación electrónica a obras sociales, backups de DB o exports desde el navegador (`R8` la prohíbe para datos del consultorio).
- Nueva migración o retención legal distinta de los 5 años ya adoptados (supuesto SU de la KB).

## Decisions

### D1. Un servicio de export compartido, no uno por entidad
Un módulo `application/export.py` expone `exportar_entidad(recurso, formato, tenant_id, filtros)` que arma el archivo a partir de la query del recurso; los routers sólo aportan el `Recurso` y las columnas declaradas. **Alternativa descartada**: endpoint por entidad con su lógica propia (4 reimplementaciones con riesgo de que una se desvíe del formato o del test de PHI). Es la repetición del patrón D7 de `C-13`, ya que la skill `export-sin-phi` lo estandariza.

### D2. `StreamingResponse` con generador + `yield_per` para el streaming (no `FileResponse` ni `Response` en memoria)
El archivo se genera en el momento, fila por fila: `StreamingResponse` con un generador async que consume el resultado con `yield_per(n)` de SQLAlchemy async, de modo que un export grande no materializa todas las filas en memoria (doc consultada vía Context7: `StreamingResponse` acepta async iterable y expone `headers`/`media_type`; `AsyncResult.yield_per()` habilita fetch por chunks y server-side cursors). **Alternativas descartadas**: `FileResponse` (requiere persistir un archivo temporal a disco y limpiarlo, complejidad extra sin beneficio para CSVs generados on-the-fly) y `Response` con el cuerpo completo (mata el requisito de "export grande" del Flujo 7). Evidencia: `docs/skills-mcp/evidencias/04-uso-context7-en-preparacion.md`.

### D3. Formato y filename fijados por contrato (mismos de C-13/D7)
CSV UTF-8 **con BOM**, separador `,`, decimal `.`, quoting RFC 4180; JSON con `Decimal` como string; ambos con `Content-Disposition: attachment; filename="<recurso>-<clave>.<ext>"`, donde `<clave>` es **el identificador interno o el rango** (p. ej. `turnos-202611.csv`), nunca nombre/DNI de paciente (`R9`). **Alternativa descartada**: filename con nombre del paciente "para reconocer el archivo" — el archivo viaja por email/descarga y el nombre queda en el historial de descargas del SO.

### D4. RBAC: `Accion.EXPORTAR` por recurso en la matriz (patrón D8 de C-13)
`MATRIZ[Recurso.…]` gana `EXPORTAR`: Dueño los 4 recursos + auditoría; Recepción agenda/caja/liquidaciones (fila "Exportación CSV/JSON: Parcial" de `03_actores_y_roles.md`); Odontólogo `403` en los 4 (su parcialidad es "clínica propia", no incluida en el alcance de C-14); auditoría sólo Dueño (fila "Auditoría: Dueño R, resto —"). Rol sin permiso → `403` sin revelar existencia; recurso de otro tenant → `404`; `formato` distinto de `csv|json` → `422`.

### D5. `tenant_id` y actor sólo del JWT (`R4`)
Ningún filtro de tenant viene del query/body. Los filtros de `GET /api/auditoria` (entidad, período, actor) se aplican **encima** del `tenant_id` del token, usando el índice `(tenant_id, entidad, entidad_id, created_at)`.

### D6. La descarga es un INSERT más en `auditoría` (`RN-SE-01`)
`accion = EXPORTAR`, `entidad` = recurso exportado, `entidad_id` = alcance (rango/período o id), `actor_id`, `created_at`, `antes/despues` = `null`/resumen del alcance sin PHI. El endpoint de auditoría y los exports son sólo lectura + ese INSERT: cero UPDATE/DELETE (se verifica además contra el constraint de DB de `C-02`).

### D7. Export grande → job asíncrono en RQ con descarga por ID interno (`Flujo 7`)
Umbral por cantidad de filas estimada; al superarlo la API responde `202` con `job_id` y el worker RQ (ya presente en `compose` desde `C-01`) deja el archivo para descargarlo al terminar. El ticket/log del job referencia el `job_id` y el `tenant_id`, jamás datos del paciente (`R9`). **Alternativa descartada**: export siempre síncrono (rompe con volúmenes de años de agenda) y export siempre asíncrono (penaliza el caso chico, que es el 95%).

### D8. Sin migración; interceptor de auditoría sólo si no está centralizado
El scope dice "sin migración": se leen `auditoria` y las 4 entidades ya existentes. Si algún cambio previo no dejó el INSERT centralizado, se agrega un interceptor en la capa application **sin tocar esquema**. Si la implementación requiriera migración, la spec está mal (checklist de la skill) y se levanta en review.

### D9. Página legal pública y portabilidad del paciente (`RN-SE-02`)
Ruta pública sin auth (rate-limit) con el texto de Ley 25.326 + Ley HCE, y export de los datos del propio paciente — único export que **sí** incluye identificantes, limitado a lo declarado en la spec (mínimo necesario).

## Risks / Trade-offs
- [El generador async y la session por request: la sesión podría cerrarse antes de terminar de streamear] → el generador recibe la session como dependencia de la request y `StreamingResponse` la consume dentro del ciclo de vida de la request (`R3`); si se opts por cerrarla temprano, bufferizar por chunks con `yield_per` antes de iterar.
- [Un export de pacientes concentra PHI en un archivo descargable] → sólo Dueño con `EXPORTAR`, columna explícita en la spec, filename sin PHI y fila `EXPORTAR` en auditoría que registra quién lo bajó.
- [El umbral del job asíncrono es un número sin dato real] → umbral configurable por env var con default conservador; no cambia specs (los dos caminos están en la spec).
- [Duplicidad de comportamiento entre el export de `C-13` (por liquidación) y el nuevo export de listado de liquidaciones] → el de `C-13` sigue existiendo sin cambios; el nuevo exporta el listado, no el detalle; se cubre con test para no confundir contratos.

## Migration Plan
Sin migración (lectura de tablas existentes). Rollback = quitar endpoints/routers y la fila `EXPORTAR`; ningún dato existente se muta. Si aparece un interceptor de auditoría, se revierte quitando el hook (la tabla ya tenía los eventos).

## Open Questions
- Umbral exacto de filas para el job asíncrono (se define por env var en implementación; no cambia specs ni tareas).
