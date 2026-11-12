# Evidencia 03 — uso de la skill propia `export-sin-phi` en la preparación de C-14

Fecha: 2026-11-12
Skill: `.agents/skills/export-sin-phi/SKILL.md` (skill propia del TP, creada con `skill-creator` — ver `docs/skills-mcp/investigacion.md` §2)
Secciones usadas: **Reglas del dominio**, **Procedimiento** (paso 2 RBAC/tenant, paso 3 generación en servidor, paso 4 columnas mínimas/cero PHI, paso 5 auditar la descarga, paso 6 batería de tests por entidad), **Criterios de revisión**.

## Solicitud (qué preparación se hacía)

Preparar el change `c-14-auditoria-exportacion` con el flujo SDD del proyecto (proposal, design, spec con escenarios verificables, tasks mapeadas a escenarios), partiendo de la sección `[C-14]` de `CHANGES.md` y la KB (`04` §Auditoria, `05` RN-SE-01/02, `06` US-051, `07` Flujo 7, `03` matriz RBAC). La skill se disparó por descripción: su `description` menciona literalmente "el change C-14 auditoria-exportacion" y el "1 clic del Dueño del Flujo 7/US-051".

## Qué hizo la skill (aplicación del procedimiento)

1. **Lectura obligatoria antes de "codear"** (paso 1): forzó extraer de la spec/KB columnas exactas, roles con `EXPORTAR`, formato y si alguna entidad necesita identificantes del paciente → resolvió que **solo el export de pacientes** (portabilidad `RN-SE-02`) los incluye.
2. **Contrato de endpoint** (paso 2): `GET /api/<recurso>/exportar?formato=csv|json`, tenant solo del JWT (R4), `403` sin permiso / `404` otro tenant / `422` formato inválido → traducido directo en requirements y escenarios.
3. **Formato** (paso 3): CSV UTF-8+BOM/`,`/`.`/RFC 4180, JSON con `Decimal` como string, `Content-Disposition` con filename → requirement de formato.
4. **Cero PHI** (paso 4): columnas exactas, sin DNI/teléfono/HC en agenda/caja/liquidaciones, **filename y logs sin PHI**, ticket de job por ID interno → requirements y escenarios propios.
5. **Auditar la descarga** (paso 5): fila `accion = EXPORTAR` por cada descarga, cero UPDATE/DELETE → requirement de auditoría.
6. **Batería de tests por entidad** (paso 6): tabla de 6 comportamientos × ≥2 casos (aislamiento, cero PHI, RBAC, formato, R8, auditoría) → cada fila se volcó en las tareas con archivo de test concreto.
7. **Checklist de revisión** → tarea 5.2 (`code-review` + checklist) y 5.3 (docs sin desviarse del contrato).

## Resultado / aporte concreto (antes → después)

| Artefacto | Sin la skill (lo que decía solo `CHANGES.md`) | Con la skill (lo que quedó escrito) |
|---|---|---|
| **Spec — escenarios** | El scope solo decía "Tests: matriz (recepción/odontólogo sin acceso), export contiene solo datos del tenant, auditoría inmutable" (3 ideas sueltas). | **17 escenarios** en 9 requirements, incluyendo los que la skill agregó: **filename sin PHI** ("El filename no expone datos del paciente"), **fila `EXPORTAR` por descarga** ("La descarga deja su fila EXPORTAR"), **la auditoría no cambia al exportar**, **`formato` inválido → `422` sin fila de auditoría**, **export de pacientes como único con identificantes**, **`202` + `job_id` sin PHI en logs**. |
| **Spec — RBAC** | "recepción/odontólogo sin acceso" (sin desagregar). | Filas de la matriz `03` traducidas a 2 escenarios: Recepción exporta agenda pero **no** pacientes; Odontólogo `403` en los 4; auditoría solo Dueño. |
| **Spec — formato** | No mencionaba formato (eso vivía solo en C-13). | Requirement propio con BOM/`,`/`.`/RFC 4180 y `Decimal` como string, heredado del paso 3 de la skill. |
| **Design** | No decidía streaming ni PHI en filename. | **D2** (StreamingResponse + `yield_per`, consultado en Context7) y **D3** (filename con clave/rango, alternativa "filename con nombre del paciente" explícitamente descartada por R9). |
| **Tasks** | Lista genérica de "tests". | **16 tareas sin marcar**, cada una con archivo `tests/test_<recurso>_export*.py`, **≥2 casos por comportamiento**, DB real (R11), fixtures sintéticos (`Paciente Agenda Export`, R13) y **mapeo explícito `→ Escenario: "<nombre>"`** tarea→escenario. |
| **Revisión** | — | Tarea 5.2 con el checklist literal de "Criterios de revisión" (incluida la verificación de "sin migración", propia de C-14). |

**Conclusión**: sin la skill, la spec no habría incluido el escenario de **filename con PHI** ni el de la **fila `EXPORTAR` en auditoría**, ni la desagregación RBAC por recurso ni el formato CSV heredado de C-13; las tareas habrían quedado como lista genérica sin mapeo escenario↔tarea ni el piso de ≥2 casos por comportamiento.
