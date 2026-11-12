# Spec Delta: `auditoria-exportacion`

## Purpose

Permitirle al Dueño auditar quién cambió qué y cuándo, y exportar en 1 clic agenda, pacientes, caja y liquidaciones en CSV/JSON generado en servidor, con aislamiento por tenant, permiso `EXPORTAR`, cero PHI fuera de lo declarado y cada descarga registrada en la tabla append-only (`RN-SE-01`, `RN-SE-02`, `US-051`, Flujo 7).

## ADDED Requirements

### Requirement: Auditoría consultable por entidad, período y actor
El sistema MUST exponer `GET /api/auditoria` con filtros por `entidad`, rango de fechas y `actor_id`, devolviendo el trail append-only (`quién`, `qué`, `cuándo`, `antes`, `despues`) del tenant del token (`RN-SE-01`, `R4`). El endpoint MUST estar restringido al rol Dueño (`03_actores_y_roles.md`, fila Auditoría) y MUST usar el índice `(tenant_id, entidad, entidad_id, created_at)` para los filtros.

#### Scenario: El Dueño filtra la auditoría por entidad y período
- **GIVEN** eventos de `turno` y de `caja` registrados en el tenant dentro y fuera del rango consultado
- **WHEN** el Dueño solicita `GET /api/auditoria?entidad=turno&desde=2026-11-01&hasta=2026-11-30`
- **THEN** recibe únicamente los eventos de `turno` del rango con actor, acción, valores `antes`/`despues` y `created_at`, ordenados por fecha, y ningún evento de `caja` ni de otro tenant

#### Scenario: Recepción y Odontólogo no consultan la auditoría
- **GIVEN** usuarios autenticados con rol Recepción y con rol Odontólogo
- **WHEN** ambos solicitan `GET /api/auditoria`
- **THEN** el sistema responde `403` a ambos sin devolver ningún evento ni confirmar su existencia

### Requirement: Exportación CSV/JSON generada en servidor para las cuatro entidades
El sistema MUST generar en el servidor `GET /api/<recurso>/exportar?formato=csv|json` para `agenda`, `pacientes`, `caja` y `liquidaciones`, devolviendo el archivo con `Content-Disposition: attachment; filename="<recurso>-<clave>.<ext>"` (`US-051` CA-2, `R8`). El contenido MUST calcularse exclusivamente en backend a partir de la query al tenant del token; el cliente MUST poder indicar sólo el formato (`R4`, `R8`).

#### Scenario: El Dueño exporta la agenda del mes en CSV
- **GIVEN** turnos sintéticos del tenant con inicio dentro de noviembre de 2026
- **WHEN** el Dueño solicita `GET /api/turnos/exportar?formato=csv&desde=2026-11-01&hasta=2026-11-30`
- **THEN** recibe `200` con un archivo CSV UTF-8 con BOM, separador `,`, decimal `.`, quoting RFC 4180, `Content-Disposition: attachment` y exactamente las columnas declaradas para agenda, con los mismos valores sembrados en la base de datos

#### Scenario: Caja y liquidaciones exportan en JSON con montos como string
- **GIVEN** movimientos de caja y liquidaciones del tenant con montos `Decimal`
- **WHEN** el Dueño solicita cada export con `formato=json`
- **THEN** la respuesta es JSON válido cuyos montos serializan como string con dos decimales y sin claves extra a las declaradas

#### Scenario: Formato de exportación no soportado
- **GIVEN** un recurso exportable del consultorio
- **WHEN** se solicita la exportación con `formato=xml` (o sin `formato`)
- **THEN** el sistema responde `422` y no genera archivo ni registra fila de auditoría

### Requirement: Aislamiento por tenant en cada archivo exportado
Cada archivo MUST contener sólo filas cuyo `tenant_id` sea el del token (`R4`). Un recurso que exista sólo en otro tenant MUST responder `404`, y las filas de otros tenants MUST estar ausentes del archivo cuando el recurso es coleccionable.

#### Scenario: El archivo de agenda no incluye turnos de otro consultorio
- **GIVEN** dos tenants con turnos en el mismo rango de fechas
- **WHEN** el Dueño del tenant A exporta la agenda en CSV
- **THEN** el archivo contiene sólo filas del tenant A y ninguna del tenant B

#### Scenario: Export de un recurso inexistente en el propio tenant
- **GIVEN** una liquidación registrada únicamente en el tenant B
- **WHEN** el Dueño del tenant A solicita exportarla por id
- **THEN** el sistema responde `404` sin revelar si la liquidación existe en otro tenant

### Requirement: Permiso EXPORTAR explícito por recurso en la matriz RBAC
Exportar MUST requerir el permiso `Accion.EXPORTAR` del recurso según la matriz `03_actores_y_roles.md` (`R4`, patrón D8 de `C-13`): Dueño en los 4 recursos; Recepción en agenda, caja y liquidaciones; Odontólogo con `403` en los 4 exports de este change. Un rol con permiso de lectura pero sin `EXPORTAR` MUST recibir `403`.

#### Scenario: Recepción exporta agenda pero Odontólogo recibe 403
- **GIVEN** usuarios autenticados con rol Recepción y con rol Odontólogo sobre el mismo tenant
- **WHEN** ambos solicitan `GET /api/turnos/exportar?formato=csv`
- **THEN** Recepción recibe el archivo y el Odontólogo recibe `403` sin archivo

#### Scenario: Recepción no exporta pacientes
- **GIVEN** una usuaria con rol Recepción
- **WHEN** solicita `GET /api/pacientes/exportar?formato=csv`
- **THEN** el sistema responde `403`, reservando la portabilidad de pacientes al Dueño

### Requirement: Columnas mínimas y cero PHI fuera de lo declarado
Los archivos de agenda, caja y liquidaciones MUST incluir sólo las columnas declaradas en este spec y MUST NOT contener DNI, teléfono, datos clínicos ni el nombre del paciente (`RN-SE-02`, `R9`). El filename, los logs de la descarga y los mensajes de error tampoco MUST llevar PHI. Sólo el export de pacientes (portabilidad) MUST incluir los identificantes del paciente declarados en su fila de columnas.

#### Scenario: El export de caja no contiene identificantes del paciente
- **GIVEN** movimientos de caja de turnos de pacientes con DNI, teléfono y evolución clínica registrados
- **WHEN** el Dueño exporta caja en CSV y en JSON
- **THEN** el archivo no contiene DNI, teléfono, nombre de paciente ni datos clínicos, y sus columnas son exactamente las declaradas para caja

#### Scenario: El filename no expone datos del paciente
- **GIVEN** una exportación de agenda solicitada por el Dueño
- **WHEN** se descarga el archivo
- **THEN** el `Content-Disposition` usa un filename basado en recurso y clave o rango (p. ej. `turnos-202611.csv`), sin nombre ni DNI de paciente

#### Scenario: El export de pacientes sí incluye los identificantes declarados
- **GIVEN** pacientes sintéticos del tenant (fixtures con DNI sintético, jamás datos reales)
- **WHEN** el Dueño exporta pacientes en CSV por portabilidad (`RN-SE-02`)
- **THEN** el archivo incluye exactamente las columnas declaradas de identificación y ninguna columna clínica

### Requirement: Cada descarga se registra en la auditoría append-only
Todo export exitoso MUST INSERTAR una fila en `auditoria` con `accion = EXPORTAR`, `entidad` (recurso exportado), `entidad_id` o alcance, `actor_id` y `created_at`, sin PHI en `antes`/`despues` (`RN-SE-01`, `R9`). La descarga MUST ser sólo lectura + ese INSERT: sin UPDATE ni DELETE sobre `auditoria`.

#### Scenario: La descarga deja su fila EXPORTAR
- **GIVEN** el Dueño autenticado
- **WHEN** exporta exitosamente la agenda en CSV
- **THEN** se inserta una fila `EXPORTAR` con `entidad = turno` (o el recurso equivalente), el `actor_id` del Dueño, el alcance del rango y su timestamp

#### Scenario: La auditoría no cambia al exportar
- **GIVEN** la tabla `auditoria` con N filas previas a una exportación
- **WHEN** el Dueño realiza la descarga
- **THEN** esas N filas quedan idénticas (sólo se agrega la fila `EXPORTAR`) y no se ejecuta ningún UPDATE ni DELETE

### Requirement: La auditoría es inmutable por permiso y por base de datos
Intentar borrar o modificar eventos de auditoría MUST rechazarse en la API con `403` y MUST fallar además en la base de datos por los permisos `INSERT`/`SELECT` exclusivos del rol app (`core-models-multitenant`, `RN-SE-01`), quedando los registros intactos.

#### Scenario: Intento de borrar auditoría rechazado en las dos capas
- **GIVEN** eventos de auditoría existentes en el tenant
- **WHEN** un usuario autenticado intenta eliminarlos (endpoint o sentencia con el rol app)
- **THEN** la API responde `403` y la base de datos rechaza la operación por permiso insuficiente, con los registros intactos

### Requirement: Exportación de gran volumen degrada a job asíncrono
Cuando el volumen a exportar supere el umbral configurado, el sistema MUST responder `202` con un `job_id` y encolar la generación en el worker, dejando el archivo disponible para descarga al terminar (`Flujo 7`). El ticket y los logs del job MUST identificar por `job_id` e `tenant_id`, jamás por datos del paciente (`R9`).

#### Scenario: Un export grande responde con ticket de job
- **GIVEN** un volumen de filas superior al umbral configurado para el recurso
- **WHEN** el Dueño solicita la exportación
- **THEN** el sistema responde `202` con el `job_id`, encola el trabajo y el archivo queda disponible para descarga cuando el worker termina, sin PHI en el ticket ni en los logs

### Requirement: Página legal pública de datos personales
El sistema MUST servir una ruta pública sin autenticación (con rate-limit) con la información de tratamiento de datos según Ley 25.326 y Ley HCE, incluyendo el derecho de exportación de los propios datos del paciente (`RN-SE-02`).

#### Scenario: La página legal responde sin token
- **GIVEN** un visitante sin sesión
- **WHEN** accede a la ruta pública de información legal
- **THEN** recibe la página con los textos de la ley y el mecanismo de portabilidad, sin exponer datos de ningún paciente
