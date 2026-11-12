# Proposal: `c-14-auditoria-exportacion`

## Why
El consultorio acumula cambios críticos (turnos, HC, caja) auditados en una tabla `auditoria` que desde `C-02` es append-only pero **nadie puede consultar**, y el Dueño no puede bajar sus datos en un archivo: quién cambió qué y cuándo sólo se ve por query directa a la DB, y agenda/pacientes/caja/liquidaciones sólo se ven en pantalla (`US-051` CA-1/CA-2, Flujo 7). Sin este change no hay confianza ni portabilidad, y `C-17` (admin/export en UI) queda bloqueado.

## What Changes
- **Auditoría consultable (CA-1)**: `GET /api/auditoria` con filtros por `entidad`, período y `actor`, devolviendo el trail append-only (`RN-SE-01`) del tenant del token, **solo para el rol Dueño** (matriz `03_actores_y_roles.md`).
- **Exportación 1-clic en servidor (CA-2)**: `GET /api/<recurso>/exportar?formato=csv|json` para **agenda, pacientes, caja y liquidaciones**, generando el archivo en backend (`R8`) en el formato ya validado en `C-13` (CSV UTF-8 con BOM, `,`, decimal `.`, RFC 4180; JSON con `Decimal` como string; `Content-Disposition: attachment`).
- **Columnas mínimas y cero PHI (`RN-SE-02`, `R9`)**: agenda/caja/liquidaciones sin DNI, teléfono ni datos clínicos; **solo el export de pacientes** (portabilidad) incluye los identificantes declarados. Filename y logs de la descarga tampoco llevan PHI.
- **Permiso `EXPORTAR` explícito por recurso** (patrón D8 de `C-13`): Dueño los 4 exports; Recepción agenda/caja/liquidaciones; Odontólogo `403` en los 4 exports de este change; auditoría sólo Dueño.
- **La descarga se audita**: cada export exitoso INSERTA una fila `accion = EXPORTAR` en `auditoria`; la tabla sigue siendo sólo INSERT+SELECT (rechazo de UPDATE/DELETE por permiso + constraint, con test explícito).
- **Exportación grande → job asíncrono** con descarga al terminar, identificando el ticket por ID interno y nunca por datos del paciente (Flujo 7 casos de error).
- **Página legal pública** (Ley 25.326 + Ley HCE) y exportación de los propios datos del paciente (`RN-SE-02`).
- **Sin migración**: lee las tablas existentes (`auditoria` desde `C-02`, exports sobre entidades de `C-06`/`C-05`/`C-11`/`C-13`); sólo si el "middleware/interceptor de auditoría" no está centralizado se agrega sin tocar el esquema.

## Capabilities

### New Capabilities
- `auditoria-exportacion`: trail de auditoría consultable con filtros y exportación CSV/JSON en servidor de agenda, pacientes, caja y liquidaciones, con aislamiento por tenant, RBAC `EXPORTAR`, columnas mínimas sin PHI, registro `EXPORTAR` de cada descarga y job asíncrono para volúmenes grandes (`RN-SE-01`, `RN-SE-02`, `US-051`, Flujo 7).

### Modified Capabilities
- Ninguna. `core-models-multitenant` ya garantiza la inmutabilidad de `auditoria` (INSERT/SELECT) y `obras-sociales-liquidacion` ya especifica el export **por liquidación** (`GET /{id}/exportar`); este change agrega el export de listados por recurso bajo la nueva capability sin alterar esos contratos vigentes.

## Impact
- **Governance**: `CRITICO` (auditoría + PHI + reglas `RN-SE`); el TP sólo prepara el change, no lo implementa.
- **Dependencias**: `C-11 caja-senas` y `C-13 obras-sociales-liquidacion` — ambas archivadas con specs vigentes.
- **Código afectado (al implementar)**: `src/backend/app/api/routers/` (nuevo `auditoria.py` + endpoints `exportar` en los routers de turnos/pacientes/caja/liquidaciones), `src/backend/app/application/` (servicio de export compartido), `src/backend/app/domain/rbac.py` (`Accion.EXPORTAR` por recurso), `src/frontend/` (página legal + botón de export en `C-17`, fuera de este change salvo la página legal), `tests/` (`test_auditoria_api.py`, `tests/test_<recurso>_export.py`).
- **Migraciones**: ninguna (lectura de tablas existentes).
- **Desbloquea**: `C-17 frontend-caja-os-admin` (auditoría y export en la UI del admin).
