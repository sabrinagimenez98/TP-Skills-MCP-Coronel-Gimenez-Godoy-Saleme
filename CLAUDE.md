# Turnos Consultorio Odontología — Instrucciones para Agentes

> Reglas globales ya definidas en `~/.claude/CLAUDE.md` (orquestador, governance, TDD, engram): el proyecto las hereda. Acá viven solo las reglas **específicas de este proyecto** + las universales que el global no cubra.

## Stack Tecnológico

| Capa | Tecnologías | Versión mínima |
|---|---|---|
| Frontend | React + TypeScript + Vite (+ Vite PWA plugin) | React 18, TS 5, Vite 5 |
| Backend | Python FastAPI + SQLAlchemy | Python 3.12, FastAPI 0.110+ |
| Base de datos | PostgreSQL (aislación por `tenant_id`; RLS opcional a futuro) | PostgreSQL 15 |
| Colas / caché | Redis + worker (RQ/Celery o ARQ) | Redis 7 |
| Auth | JWT access + refresh con rotación | — |
| Infra local | Docker + Docker Compose | Docker 24 / Compose v2 |
| Pagos | Mercado Pago SDK + webhooks (Checkout/Links/QR) | API v1 vigente |
| Mensajería | WhatsApp Business API oficial + SMTP transaccional | Cloud API v18+ |
| Exportación | CSV/JSON generados en backend | — |

Sin cloud obligatoria: Docker Compose levanta todo en local primero (piloto con 1 consultorio).

## Base de Conocimiento

Fuente de verdad del dominio. Leer antes de proponer o implementar:

- `knowledge-base/01_vision_y_objetivos.md` — propósito, alcance v1, fuera de alcance
- `knowledge-base/02_descripcion_general.md` — stack, arquitectura, API REST
- `knowledge-base/03_actores_y_roles.md` — 4 roles + matriz RBAC + rutas públicas
- `knowledge-base/04_modelo_de_datos.md` — 18 entidades, ERD, `tenant_id` en todo
- `knowledge-base/05_reglas_de_negocio.md` — reglas RN-AG/LE/PA/OS/CL/CO/SE
- `knowledge-base/06_funcionalidades.md` — épicas 1–7 MVP + épica 8 (v2)
- `knowledge-base/07_flujos_principales.md` — 7 flujos end-to-end
- `knowledge-base/08_arquitectura_propuesta.md` — patrones, directorios, env vars
- `knowledge-base/09_decisiones_y_supuestos.md` — decisiones DD + supuestos SU
- `knowledge-base/10_preguntas_abiertas.md` — PO-01..PO-04 abiertas (hosting, costo WA, retención legal, pricing)
- `knowledge-base/11_pagos_mercadopago.md` — webhooks, idempotencia, conciliación
- `discovery/discovery.md` — análisis competitivo (18 sistemas)

## Skills Disponibles

| Agente | Rol | Skills que carga |
|--------|-----|------------------|
| Backend Core | FastAPI + modelos multi-tenant | `python-fastapi-development`, `supabase-postgres-best-practices`, `tdd` |
| Backend Aux | Auth, pagos, mensajería, auditoría | `auth-implementation-patterns`, `stripe-integration` (plantilla MP), `clerk-webhooks` (patrón webhook), `upstash-redis-js`, `hipaa-compliance`, `export-sin-phi`, `tdd`, `code-review` |
| Frontend | PWA paciente + agenda/HC | `vercel-react-best-practices`, `frontend-design`, `tdd` |
| Clínico | HC + odontograma FDI | `healthcare-emr-patterns`, `hipaa-compliance`, `frontend-design` |
| Infra | Docker + foundation | `multi-stage-dockerfile`, `python-fastapi-development` |
| Diagnóstico | WhatsApp en producción | `observe-whatsapp` |
| Calidad | Reviews HIGH/CRITICAL | `code-review` |

> Los compact rules de cada skill los resuelve el orquestador desde `.agents/skill-registry.md` (versionado en el repo, junto a las skills en `.agents/skills/`).
>
> **`export-sin-phi`** (propia): se carga en **Backend Aux** para toda tarea de exportación/descarga CSV/JSON o auditoría consultable (C-14 primero, "bajar un CSV", "exportar en 1 clic"). Necesita la spec del change en `openspec/`, la KB (`04` §Auditoria, `05` RN-SE-01/02, `07` Flujo 7) y el patrón de referencia `tests/test_liquidaciones_export.py`. El resultado se revisa con `code-review` (ejes Standards/Spec) + el checklist de la skill: cero PHI en archivo/logs/URLs, columnas mínimas declaradas, `tenant_id` solo del token, nada calculado en cliente, fila `EXPORTAR` en `auditoría`, ≥2 tests por comportamiento contra DB real.

## Integración MCP (Context7)

Servidor MCP remoto configurado en `opencode.json` (`mcp.servers.context7` → `https://mcp.context7.com/mcp`, sin secretos). **No es una skill**: es una integración que da acceso a documentación externa.

- **Propósito**: consultar documentación oficial y **versionada** de librerías del stack (FastAPI, SQLAlchemy, Pydantic, React/Vite/PWA, pytest, Alembic) con las herramientas de solo lectura `resolve-library-id` y `query-docs`.
- **Cuándo usarlo**: cuando haya que implementar o revisar código cuya API dependa de esas librerías y el conocimiento de entrenamiento pueda estar desactualizado — p. ej. preparar/implementar C-14 (`StreamingResponse`/`Response` para los exports). No reempla leer la KB ni las specs del change: primero spec/KB, luego Context7 para la API puntual.
- **Límites**: las queries salen a un servicio externo → **nunca** enviar PHI, DNI, teléfonos, montos ni datos de pacientes (R9/R13); solo nombres de librería y tema. La API key opcional va en variable de entorno, jamás en el repo (R10). Detalle de configuración, comprobación y troubleshooting en `README.md`; evidencias en `docs/skills-mcp/evidencias/`.

## Roadmap de Changes

`CHANGES.md` — 17 changes en 7 fases, camino crítico de 9:

`C-01 foundation → C-02 modelos core → C-03 auth/RBAC → C-05 pacientes → C-06 agenda → C-09 HC/odontograma → C-11 caja/señas → C-12 Mercado Pago → C-15 reserva pública PWA`

Primer change: `/opsx:propose C-01-foundation-setup`

## Reglas Duras (específicas del proyecto)

**Backend (Python/FastAPI)**
- R1. NUNCA exponer modelos SQLAlchemy crudos → schemas Pydantic request/response explícitos, `extra='forbid'` en inputs
- R2. Type hints obligatorios; `ruff` + `mypy` limpios por change
- R3. Async end-to-end con session por request; NUNCA bloquear el event loop con I/O sync
- R4. El `tenant_id` del token JWT manda → NUNCA confiar en el `tenant_id` del body/query

**Frontend (React/TS PWA)**
- R5. Prohibido `any`; `tsconfig` estricto; componentes en `PascalCase`
- R6. NUNCA datos clínicos (HC/odontograma) en `localStorage`

**Salud + pagos (CRITICAL)**
- R7. NUNCA hard delete en HC/turnos/caja → soft delete + evolución + auditoría append-only
- R8. NUNCA montos calculados en el cliente; cobros y webhooks (MP/WA) idempotentes con dedup por event-id
- R9. NUNCA PHI (DNI, tel, datos clínicos) en logs, errores visibles, URLs ni prompts

**Proceso**
- R10. Commits en formato convencional (`feat:`, `fix:`, `chore(engram):`…) y NUNCA commitear `.env`/secretos
- R11. Tests de backend contra DB real en contenedor (sin mocks de DB); mínimo 2 casos por comportamiento
- R12. Pushear a GitHub en cada cambio
- R13. NUNCA almacenar datos reales de pacientes en el repositorio ni en los ejemplos → fixtures sintéticos siempre

## Flujo de Trabajo

```text
KB → CHANGES.md → /opsx:propose → /opsx:apply → /opsx:archive → push a GitHub
```

1. Identificá el change en `CHANGES.md` y leé su sección `Leer antes`.
2. `/opsx:propose <change>` — artefactos (proposal, design, tasks).
3. `/opsx:apply <change>` — implementación con TDD estricto.
4. `/opsx:archive <change>` — sync de specs + cierre.
5. Push a GitHub en cada cambio (R12).
