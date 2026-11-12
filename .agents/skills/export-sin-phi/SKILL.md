---
name: export-sin-phi
description: Genera endpoints de exportación CSV/JSON en servidor y su batería de tests para el SaaS odontológico, cumpliendo R7/R8/R9/R11/R13 desde el primer commit. Usala SIEMPRE que haya que implementar o revisar exportación, descarga o portabilidad de datos del consultorio — agenda, pacientes, caja, liquidaciones o auditoría — el "1 clic" del Dueño del Flujo 7/US-051, o el change C-14 auditoria-exportacion; aunque solo digan "bajar un CSV", "exportar a Excel", "descargar los turnos del mes" o "ver la auditoría en un archivo". NO la uses para exportar desde el cliente (JS arma el CSV) ni para backups/exports de base de datos.
metadata:
  version: "1.0.0"
  origin: TP Skills+MCP — creada con skill-creator (Anthropic) para C-14
---

# Exportación sin PHI (CSV/JSON en servidor)

Genera endpoints de exportación que arman el archivo **en el backend** y lo devuelven como descarga, junto con la batería de tests que garantiza aislamiento por tenant, cero PHI y RBAC — de modo que cada entidad nueva a exportar sea una repetición del patrón validado, no una reimplementación.

## Propósito

C-14 (`auditoria-exportacion`) pide exportación 1-clic de **cuatro entidades distintas** (agenda, pacientes, caja, liquidaciones) más auditoría consultable (US-051: CA-1 append-only por entidad/período, CA-2 CSV/JSON). El patrón ya está probado una vez en C-13 (`tests/test_liquidaciones_export.py`); esta skill lo estandariza para no reimplementarlo a mano — con las mismas garantías — por cada entidad.

## Cuándo se activa

- Implementar cualquier endpoint `GET …/exportar?formato=csv|json` (C-14 primero; cualquier change que agregue descargas).
- Consultar/exportar la tabla `auditoría` (Flujo 7: filtra por entidad/período/actor y exporta).
- Revisar o testear un endpoint de export ya existente.
- Tareas que dicen "exportar", "bajar CSV", "descargar datos", "portabilidad", "exportación en 1 clic".

No aplica: CSV armado en el navegador, exports de infraestructura (dump de DB), o conversions de archivos — eso no pasa por esta skill.

## Reglas del dominio que gobiernan cada export (leer, no adivinar)

Citá estas reglas en el diseño/tests; son la fuente independiente contra la que se valida el resultado (TDD):

- **RN-SE-01** (`knowledge-base/05_reglas_de_negocio.md`): auditoría **append-only** — todo cambio registra quién/qué/cuándo/antes/después; **sin UPDATE ni DELETE** sobre la tabla. La descarga misma se audita como un INSERT más; nunca muta lo que exporta.
- **RN-SE-02** (ibíd.): datos de salud bajo Ley 25.326 + Ley HCE — **control de acceso por rol**, trazabilidad y **mínimo necesario** en la exportación.
- **KB `04_modelo_de_datos.md` § Auditoria**: atributos `tenant_id, actor_id, accion, entidad, entidad_id, antes, despues, ip, created_at`; constraint sin UPDATE/DELETE (solo INSERT+SELECT), retención mínima 5 años; índice (`tenant_id`, `entidad`, `entidad_id`, `created_at`) — usá ese índice para los filtros del listado.
- **KB `07_flujos_principales.md` Flujo 7**: actor **Dueño**; exporta agenda/pacientes/caja/liquidaciones CSV/JSON en 1 clic. Casos de error: export grande → job asíncrono con descarga al terminar; intento de borrar auditoría → rechazado por permiso + constraint.
- **Matriz RBAC `03_actores_y_roles.md`**: la exportación es un **permiso explícito** por recurso (patrón D8 de C-13: `Accion.EXPORTAR` en `src/backend/app/domain/rbac.py`) — Odontólogo y, según la fila de la entidad, Recepción pueden tener `403` aunque tengan lectura.
- **Reglas duras `AGENTS.md`**: R4 (el `tenant_id` **solo** del token JWT), R7 (nunca hard delete — el export solo lee), R8 (nada calculado en el cliente), R9 (cero PHI en logs/errores/URLs), R11 (tests contra DB real, ≥2 casos por comportamiento), R13 (fixtures sintéticos).

## Entradas

1. **Spec del change**: `openspec/changes/<change>/{proposal,design,tasks}.md` + specs vigentes en `openspec/specs/` — define columnas, roles con `EXPORTAR` y formato.
2. **KB relevante**: las secciones citadas arriba.
3. **Patrón de referencia**: `tests/test_liquidaciones_export.py` y el design de C-13 (`openspec/changes/archive/2026-11-05-c-13-obras-sociales-liquidacion/design.md`, decisiones D7/D8) — copiá la estructura, no la reimplementes.
4. **Entidades a exportar** y su estructura en `knowledge-base/04_modelo_de_datos.md`.

## Procedimiento

### 1. Lee antes de codear
Spec del change + las secciones de KB citadas. Extraé: columnas exactas, roles con permiso `EXPORTAR`, formato exigido, y si la spec requiere identificantes del paciente (solo el export de pacientes por portabilidad debería).

### 2. Endpoint bajo RBAC, tenant solo del token
`GET /api/<recurso>/exportar?formato=csv|json` en `src/backend/app/api/routers/`. El `tenant_id` y el actor salen del JWT (R4) — nunca de query/body. Permiso `EXPORTAR` en la matriz `MATRIZ[Recurso.…]` de `rbac.py`; rol sin permiso → `403`; recurso de otro tenant → `404`; `formato` distinto de `csv|json` → `422`.

### 3. Generación en servidor (R8)
El contenido se arma en backend a partir de la query — el cliente solo envía `?formato=`. Nunca montos, totales ni columnas calculadas en el frontend. Formato según C-13/D7: CSV **UTF-8 con BOM**, separador `,`, decimal `.`, quoting RFC 4180; JSON con `Decimal` serializado como string; ambos con `Content-Disposition: attachment; filename="<recurso>-<clave>.<ext>"`. Async end-to-end (`StreamingResponse`/`Response`, session por request — R3).

### 4. Columnas mínimas y cero PHI fuera de lo declarado (R9/RN-SE-02)
Las columnas son **exactamente** las de la spec — mínimo necesario. Para entidades que no necesitan identificar al paciente (liquidaciones, caja), el archivo **no puede** contener DNI, teléfono ni datos clínicos (trade-off ya adoptado en C-13). El `filename`, los logs de la descarga y los mensajes de error tampoco llevan PHI. Si un volumen grande obliga a job asíncrono (Flujo 7), el ticket/log del job identifica por ID interno, nunca por datos del paciente.

### 5. Auditar la descarga (RN-SE-01)
Cada export exitoso INSERTA en `auditoria`: `accion = EXPORTAR`, `entidad`, `entidad_id` (o alcance), `actor_id`, `created_at`. Nunca UPDATE/DELETE sobre `auditoria`; el endpoint de consulta de auditoría también es solo lectura append-only.

### 6. Batería de tests — por cada entidad
Archivo `tests/test_<recurso>_export.py` (o apartado en el test del recurso), contra **DB real en contenedor** (R11), fixtures **sintéticos** (R13 — un paciente de seed tipo `Paciente <recurso> Export` con DNI sintético, jamás datos reales). Mínimo 2 casos por comportamiento:

| Comportamiento | Casos mínimos |
|---|---|
| Aislamiento tenant | export del tenant propio OK; recurso de otro tenant → `404` (o ausente del archivo) |
| Cero PHI | archivo no contiene DNI/teléfono/nombre/datos clínicos (grep sobre el CSV/JSON); columnas == las declaradas |
| RBAC | rol con `EXPORTAR` → `200`; rol sin permiso (p. ej. Odontólogo) → `403` |
| Formato | CSV con BOM/`,`/RFC 4180 y `Content-Disposition`; JSON válido con `Decimal` como string; `formato` inválido → `422` |
| R8 | valores del archivo idénticos a los sembrados en DB; ningún parámetro de monto del cliente altera el resultado |
| Auditoría | la descarga insertó su fila `EXPORTAR` en `auditoria`; intento de borrar auditoría → `403` + constraint |

### 7. Quality gates
`ruff` + `mypy` limpios (R2), pytest verdes contra el contenedor, sin secretos ni PHI en el diff (R10).

## Resultado esperado

- Endpoint `GET …/exportar?formato=csv|json` funcional: `200` archivo con `Content-Disposition`, `403` rol sin permiso, `404` otro tenant, `422` formato inválido.
- Fila `EXPORTAR` insertada en `auditoria` por cada descarga.
- ≥2 tests por comportamiento verdes, contra DB real, con fixtures sintéticos y cero PHI en los archivos de prueba.
- `ruff`/`mypy` limpios; diff sin cálculos en cliente.

## Criterios de revisión

Ejecutá `code-review` (ejes Standards y Spec, por separado) y verificá además:

- [ ] Ni DNI, teléfono, nombre de paciente ni datos clínicos en el archivo exportado (cuando la spec no los declara), en logs, en URLs ni en `filename` (R9).
- [ ] Columnas exactas de la spec — sin "de regalo" (RN-SE-02 mínimo necesario).
- [ ] `tenant_id` tomado solo del token; ningún filtro de tenant venga del request (R4).
- [ ] Nada calculado en el frontend: el servidor arma el contenido completo (R8).
- [ ] Export es solo lectura + INSERT de auditoría; cero UPDATE/DELETE (R7/RN-SE-01).
- [ ] ≥2 casos por comportamiento, DB real sin mocks, fixtures sintéticos (R11/R13).
- [ ] Spec: cada criterio de aceptación del change (US-051 CA-1/CA-2 para C-14) tiene su test; `README.md`/docs no desvían del contrato real (contrato verificable con `tests/test_*_documentacion.py` si existe).
- [ ] Rollback/migración: C-14 es "sin migración (lee tablas existentes)" — si tu implementación requirió migración, la spec está mal o se desvió; levantalo en review.
