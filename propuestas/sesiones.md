# Servicio de Sesiones: decisiones, migración y contrato

Propuesta para el equipo de TutorMatch (16-11-2026). Está escrita para que la lea una persona o una IA sin más contexto: resume lo acordado sobre la agenda de tutorías y propone la migración de PostgreSQL y los cambios a `sesiones.proto`.

A diferencia de Usuarios y Material, `sesiones.proto` ya está implementado y en uso (el BFF lo consume). Por eso esta propuesta lo **modifica** en `v1` respetando las reglas del README: no reutiliza números de campo y marca como `reserved` los que elimina. Complementa a `propuestas/usuarios.md` y `propuestas/material.md`.

## Contexto mínimo

- **TutorMatch:** plataforma donde estudiantes reservan tutorías con otros estudiantes que aprobaron el ramo. Microservicios en Kubernetes.
- **Sesiones** (Spring Boot + PostgreSQL, Vicente) es dueño de la disponibilidad de los tutores y del ciclo de vida de una sesión: `PENDIENTE → CONFIRMADA → FINALIZADA`, o `CANCELADA`.
- Cómo se conecta:

| Tramo | Protocolo | Para qué |
|---|---|---|
| BFF → Sesiones | gRPC | Todo lo que pide el front: disponibilidad, reservas y su ciclo de vida. |
| Agente → Sesiones | gRPC | Ver disponibilidad y crear reservas desde el chat. |
| Material → Sesiones | gRPC | `VerificarParticipacion`, antes de aceptar una evaluación. |
| Sesiones → Usuarios | gRPC | `ValidarTutorRamo`, al marcar disponibilidad y al crear una reserva. |
| Sesiones → Material | Kafka | `sesion.creada`, `sesion.finalizada` y `sesion.cancelada` (formato en `eventos/`). |

- El front nunca llama a Sesiones directamente. El BFF verifica el JWT y pone el id del usuario en cada request, así que `tutor_id`, `estudiante_id` y `usuario_id` vienen del token, nunca del front.
- Convenciones generales: ids en UUID como texto, fechas en UTC, errores como códigos de estado gRPC.

## Bloques horarios

Las casillas de disponibilidad son los bloques de clase de la universidad, de 90 minutos, de lunes a viernes, en hora de Chile (`America/Santiago`):

| Bloque | Horario |
|---|---|
| A | 08:10 – 09:40 |
| B | 09:55 – 11:25 |
| C | 11:40 – 13:10 |
| — | Almuerzo |
| D | 14:30 – 16:00 |
| E | 16:15 – 17:45 |
| F | 18:00 – 19:30 |

- Los contratos y la base de datos usan UTC. Sesiones convierte a hora de Chile **solo para validar** que una casilla empieza en uno de estos horarios, con la zona `America/Santiago`, que considera el cambio de horario.
- El front dibuja la grilla con esta tabla y convierte cada bloque de la semana a UTC antes de llamar al BFF.
- El agente usa la misma tabla para proponer horarios.

## Decisiones acordadas

1. **Disponibilidad semana a semana y por ramo.** El tutor marca casillas con fecha concreta, cada una para un ramo que imparte. Una casilla ocupa un bloque y no puede solaparse con otra del mismo tutor, aunque sea de otro ramo.
2. **Solo bloques válidos y futuros.** Una casilla empieza en el inicio de un bloque (tabla anterior), en día hábil, dura exactamente 90 minutos y todavía no ha empezado.
3. **Para marcar, el tutor debe tener ese ramo activo** (`Usuarios.ValidarTutorRamo`).
4. **Reservar es tomar una casilla libre.** `CrearSesion` exige que exista una casilla de ese tutor, ese ramo y ese horario exacto, sin una sesión `PENDIENTE` o `CONFIRMADA` encima. Por eso toda sesión dura un bloque. También se valida con `ValidarTutorRamo`: si el tutor quitó el ramo, no se aceptan reservas nuevas.
5. **Nadie se reserva a sí mismo, y nadie tiene dos sesiones activas que se crucen**, ni como alumno ni como tutor, ni mezclando ambos papeles.
6. **Confirmación:** la reserva queda `PENDIENTE` hasta que el tutor la confirme. Rechazar es cancelar. Mientras está pendiente, la casilla ya no está libre.
7. **Desmarcar:** solo el tutor dueño, y solo si la casilla no tiene encima una sesión `PENDIENTE` o `CONFIRMADA`. Primero se cancela la sesión.
8. **Aviso al tutor sin websockets:** el front consulta las reservas pendientes del tutor cada 30 a 60 segundos. Sesiones solo necesita filtrar `ListarSesiones` por estado.
9. **Fin y vencimiento automáticos.** Una tarea programada, cada 5 minutos:
   - Pasa a `FINALIZADA` las sesiones `CONFIRMADA` cuyo `fin` ya pasó, y publica `sesion.finalizada`.
   - Pasa a `CANCELADA` las sesiones `PENDIENTE` cuyo `inicio` ya pasó sin confirmación, y publica `sesion.cancelada`.

   Así el alumno siempre puede evaluar, porque Material exige que la sesión esté `FINALIZADA`.
10. **`VerificarParticipacion` devuelve también el tutor y el ramo**, para que Material guarde la evaluación con una sola llamada (decisión de `propuestas/material.md`).

## Flujos

| Caso | RPC de Sesiones |
|---|---|
| Tutor marca una casilla en su grilla | `MarcarCasilla` |
| Tutor desmarca una casilla | `DesmarcarCasilla` |
| Tutor ve su semana | `ObtenerDisponibilidad` (sus casillas, con `libre`) + `ListarSesiones` (sus reservas en ese rango) |
| Tutor ve sus reservas por confirmar (polling) | `ListarSesiones` con `tutor_id` y `estado = PENDIENTE` |
| Tutor confirma o rechaza | `ConfirmarSesion` / `CancelarSesion` |
| Alumno busca horarios de un ramo | `BuscarDisponibilidad` (solo casillas libres) |
| Alumno ve la semana de un tutor | `ObtenerDisponibilidad` (filtrando `libre`) |
| Alumno reserva | `CrearSesion` |
| Alumno ve y cancela sus sesiones | `ListarSesiones` con `estudiante_id` / `CancelarSesion` |
| Material valida una evaluación | `VerificarParticipacion` |

## Migración V3 (PostgreSQL)

Las migraciones V1 (`btree_gist`) y V2 (`sesiones` y la `disponibilidad` vieja) ya existen. La V3 reemplaza la disponibilidad, que hasta ahora era un patrón semanal sin ramo y solo tenía datos de prueba, y agrega dos restricciones a `sesiones`:

```sql
-- La disponibilidad pasa de un patrón semanal sin ramo a casillas con fecha y ramo.
-- La tabla anterior solo tenía datos de prueba.
DROP TABLE disponibilidad;

CREATE TABLE disponibilidad (
    id          UUID        PRIMARY KEY,
    tutor_id    TEXT        NOT NULL,
    ramo_codigo TEXT        NOT NULL,
    inicio      TIMESTAMPTZ NOT NULL,
    fin         TIMESTAMPTZ NOT NULL,
    horario     TSTZRANGE   NOT NULL GENERATED ALWAYS AS (tstzrange(inicio, fin, '[)')) STORED,
    -- Una casilla es un bloque de clase. Que empiece en un bloque válido (hora de Chile)
    -- lo valida el servicio, porque depende de la zona horaria.
    CHECK (fin = inicio + interval '90 minutes'),
    -- Un tutor no puede tener dos casillas que se solapen, aunque sean de ramos distintos.
    CONSTRAINT disponibilidad_sin_solape
        EXCLUDE USING gist (tutor_id WITH =, horario WITH &&)
);

-- Nadie se reserva a sí mismo.
ALTER TABLE sesiones ADD CONSTRAINT sesiones_no_a_si_mismo CHECK (tutor_id <> estudiante_id);

-- Un alumno no puede tener dos sesiones activas que se crucen (la de los tutores ya existe en V2).
ALTER TABLE sesiones ADD CONSTRAINT sesiones_estudiante_sin_choque
    EXCLUDE USING gist (estudiante_id WITH =, horario WITH &&)
    WHERE (estado IN ('PENDIENTE', 'CONFIRMADA'));
```

### Lo que valida el servicio y no la base

- **Inicio en un bloque válido:** la base no conoce la hora de Chile, así que lo valida Java:

  ```java
  private static final ZoneId CHILE = ZoneId.of("America/Santiago");
  private static final Set<LocalTime> INICIOS_DE_BLOQUE = Set.of(
          LocalTime.of(8, 10), LocalTime.of(9, 55), LocalTime.of(11, 40),
          LocalTime.of(14, 30), LocalTime.of(16, 15), LocalTime.of(18, 0));

  static boolean esInicioDeBloque(Instant inicio) {
      ZonedDateTime local = inicio.atZone(CHILE);
      return local.getDayOfWeek().getValue() <= 5 && INICIOS_DE_BLOQUE.contains(local.toLocalTime());
  }
  ```

- **Cruce entre papeles** (decisión 5): las restricciones `EXCLUDE` cubren tutor con tutor y alumno con alumno, pero no "el tutor de una sesión es alumno en otra a la misma hora". Antes de crear, el servicio busca sesiones activas de **ambos** participantes que se crucen:

  ```sql
  SELECT EXISTS (
      SELECT 1 FROM sesiones
      WHERE estado IN ('PENDIENTE', 'CONFIRMADA')
        AND horario && tstzrange(:inicio, :fin, '[)')
        AND (tutor_id IN (:tutor, :estudiante) OR estudiante_id IN (:tutor, :estudiante))
  );
  ```

  Dos reservas simultáneas podrían colarse entre la consulta y el `INSERT`. Se acepta para este proyecto: los casos más comunes ya los cubren los `EXCLUDE`.

- **Casilla libre:** una casilla está libre si no hay una sesión `PENDIENTE` o `CONFIRMADA` del mismo tutor que se cruce con ella:

  ```sql
  SELECT d.*, NOT EXISTS (
      SELECT 1 FROM sesiones s
      WHERE s.tutor_id = d.tutor_id
        AND s.estado IN ('PENDIENTE', 'CONFIRMADA')
        AND s.horario && d.horario
  ) AS libre
  FROM disponibilidad d
  WHERE d.tutor_id = :tutor AND d.inicio >= :desde AND d.inicio < :hasta
  ORDER BY d.inicio;
  ```

- **Ramos inactivos en la búsqueda:** si un tutor quitó un ramo, sus casillas de ese ramo no se borran. `BuscarDisponibilidad` llama a `ValidarTutorRamo` una vez por cada tutor distinto del resultado y omite a los que ya no tienen el ramo activo.

## Contrato `sesiones.proto`

```proto
// Contrato gRPC del servicio de Sesiones.
// Lo implementa Sesiones (Spring Boot, Vicente) y lo consumen el BFF, Material y el agente.
//
// Disponibilidad: el tutor marca casillas con fecha concreta, cada una para un ramo. Una
// casilla es un bloque de clase de 90 minutos, de lunes a viernes, que empieza a una de
// estas horas de Chile (America/Santiago): 08:10, 09:55, 11:40, 14:30, 16:15 o 18:00.
// Los Timestamp viajan en UTC; Sesiones convierte a hora de Chile solo para validar.
//
// Una reserva (sesión) ocupa exactamente una casilla libre: misma fecha, tutor y ramo, sin
// otra sesión PENDIENTE o CONFIRMADA encima.
//
// Ciclo de vida: PENDIENTE -> CONFIRMADA -> FINALIZADA, o CANCELADA. Una tarea programada
// finaliza las CONFIRMADA cuyo fin pasó y cancela las PENDIENTE cuyo inicio pasó.
//
// tutor_id, estudiante_id y usuario_id los pone el BFF a partir del JWT.
//
// Dependencias de Sesiones: Usuarios.ValidarTutorRamo al marcar una casilla y al crear una sesión.
//
// Los errores viajan como códigos de estado gRPC, no como campos de la respuesta:
//   NOT_FOUND            la sesión o la casilla no existe.
//   ALREADY_EXISTS       la casilla se solapa con otra del mismo tutor, o alguno de los dos
//                        participantes ya tiene una sesión activa en ese horario.
//   FAILED_PRECONDITION  el tutor no tiene ese ramo activo; no hay una casilla libre para ese
//                        horario; la transición de estado no es válida; la casilla que se
//                        quiere desmarcar tiene una sesión PENDIENTE o CONFIRMADA.
//   PERMISSION_DENIED    el usuario no participa en la sesión, o la casilla no es suya.
//   INVALID_ARGUMENT     faltan datos; el horario no es un bloque válido o ya pasó; alguien
//                        intenta reservarse a sí mismo.
//   UNAVAILABLE          Usuarios no responde.
syntax = "proto3";

package tutorias.sesiones.v1;

import "google/protobuf/timestamp.proto";

option java_multiple_files = true;
option java_package = "cl.tutorias.sesiones.grpc";

service SesionesService {
  rpc CrearSesion(CrearSesionRequest) returns (Sesion);
  rpc ObtenerSesion(ObtenerSesionRequest) returns (Sesion);
  rpc ListarSesiones(ListarSesionesRequest) returns (ListarSesionesResponse);
  // PENDIENTE -> CONFIRMADA. Solo el tutor de la sesión.
  rpc ConfirmarSesion(CambiarEstadoRequest) returns (Sesion);
  // PENDIENTE o CONFIRMADA -> CANCELADA. El tutor (rechazar) o el alumno.
  rpc CancelarSesion(CambiarEstadoRequest) returns (Sesion);
  // CONFIRMADA -> FINALIZADA. Normalmente lo hace la tarea programada; el tutor puede adelantarlo.
  rpc FinalizarSesion(CambiarEstadoRequest) returns (Sesion);

  // El tutor marca una casilla para un ramo.
  rpc MarcarCasilla(MarcarCasillaRequest) returns (BloqueDisponibilidad);
  // El tutor desmarca una de sus casillas sin sesión activa encima.
  rpc DesmarcarCasilla(DesmarcarCasillaRequest) returns (DesmarcarCasillaResponse);
  // Casillas de un tutor en un rango, libres u ocupadas (campo libre).
  rpc ObtenerDisponibilidad(ObtenerDisponibilidadRequest) returns (ObtenerDisponibilidadResponse);
  // Casillas libres de todos los tutores de un ramo en un rango, para el buscador del alumno.
  rpc BuscarDisponibilidad(BuscarDisponibilidadRequest) returns (ObtenerDisponibilidadResponse);

  // Usado por Material antes de aceptar una evaluación.
  rpc VerificarParticipacion(VerificarParticipacionRequest) returns (VerificarParticipacionResponse);
}

enum EstadoSesion {
  // proto3 exige un valor 0; nunca se guarda ni se devuelve. En los filtros significa "todos".
  ESTADO_SESION_DESCONOCIDO = 0;
  PENDIENTE = 1;
  CONFIRMADA = 2;
  FINALIZADA = 3;
  CANCELADA = 4;
}

message Sesion {
  string id = 1;
  string tutor_id = 2;
  string estudiante_id = 3;
  string ramo_codigo = 4;
  // Rango [inicio, fin): siempre un bloque de 90 minutos.
  google.protobuf.Timestamp inicio = 5;
  google.protobuf.Timestamp fin = 6;
  EstadoSesion estado = 7;
  google.protobuf.Timestamp creada_en = 8;
}

// Debe coincidir con una casilla libre del tutor para ese ramo y ese horario.
message CrearSesionRequest {
  string tutor_id = 1;
  string estudiante_id = 2;
  string ramo_codigo = 3;
  google.protobuf.Timestamp inicio = 4;
  google.protobuf.Timestamp fin = 5;
}

message ObtenerSesionRequest {
  string id = 1;
}

// Todos los filtros son opcionales: un campo vacío no filtra.
message ListarSesionesRequest {
  string tutor_id = 1;
  string estudiante_id = 2;
  EstadoSesion estado = 3;
  // Sesiones que empiezan en [desde, hasta).
  google.protobuf.Timestamp desde = 4;
  google.protobuf.Timestamp hasta = 5;
}

message ListarSesionesResponse {
  repeated Sesion sesiones = 1;
}

message CambiarEstadoRequest {
  string id = 1;
  // Quién pide el cambio; debe ser el tutor o el estudiante de la sesión.
  string usuario_id = 2;
}

// Una casilla de disponibilidad: un bloque de 90 minutos para un ramo.
message BloqueDisponibilidad {
  // Eliminados: el patrón semanal (día y horas locales) se reemplazó por fechas concretas.
  reserved 2, 3, 4;
  reserved "dia", "hora_ini", "hora_fin";

  string tutor_id = 1;
  string id = 5;
  string ramo_codigo = 6;
  google.protobuf.Timestamp inicio = 7;
  google.protobuf.Timestamp fin = 8;
  // false si tiene encima una sesión PENDIENTE o CONFIRMADA.
  bool libre = 9;
}

message MarcarCasillaRequest {
  string tutor_id = 1;
  string ramo_codigo = 2;
  // Inicio de un bloque válido; el fin es siempre inicio + 90 minutos.
  google.protobuf.Timestamp inicio = 3;
}

message DesmarcarCasillaRequest {
  string id = 1;
  string tutor_id = 2;
}

message DesmarcarCasillaResponse {}

message ObtenerDisponibilidadRequest {
  string tutor_id = 1;
  // Casillas que empiezan en [desde, hasta). Obligatorios.
  google.protobuf.Timestamp desde = 2;
  google.protobuf.Timestamp hasta = 3;
}

message BuscarDisponibilidadRequest {
  string ramo_codigo = 1;
  // Casillas que empiezan en [desde, hasta). Obligatorios.
  google.protobuf.Timestamp desde = 2;
  google.protobuf.Timestamp hasta = 3;
}

message ObtenerDisponibilidadResponse {
  // Ordenadas por inicio.
  repeated BloqueDisponibilidad bloques = 1;
}

message VerificarParticipacionRequest {
  string sesion_id = 1;
  string estudiante_id = 2;
}

message VerificarParticipacionResponse {
  bool finalizada = 1;
  bool participo = 2;
  string tutor_id = 3;
  string ramo_codigo = 4;
}
```

## Qué cambia respecto al contrato actual

| Actual | Esta propuesta | ¿Rompe a alguien? |
|---|---|---|
| `BloqueDisponibilidad` con `dia`, `hora_ini`, `hora_fin` | `id`, `ramo_codigo`, `inicio`, `fin`, `libre`; los campos viejos quedan `reserved` | Al BFF, que hay que actualizar junto con Sesiones. El agente no lo usa todavía. |
| `ObtenerDisponibilidad(tutor_id)` | Agrega `desde` y `hasta`, obligatorios | Igual que el anterior. |
| — | `MarcarCasilla`, `DesmarcarCasilla`, `BuscarDisponibilidad` | No: son RPC nuevos. |
| `ListarSesiones(tutor_id, estudiante_id)` | Agrega `estado`, `desde` y `hasta`, opcionales | No: campos nuevos. |
| `VerificarParticipacionResponse(finalizada, participo)` | Agrega `tutor_id` y `ramo_codigo` | No: campos nuevos. |
| `CrearSesion` acepta cualquier horario | Exige una casilla libre de 90 minutos y valida con `ValidarTutorRamo` | No cambia el mensaje, solo agrega errores. |
| `ConfirmarSesion` lo podía hacer el alumno | Solo el tutor | Corrige una regla que faltaba. |

Este es el único cambio que rompe algo: `BloqueDisponibilidad` cambia de forma. Lo consume solo el BFF, y los dos servicios son de Vicente, así que se actualizan juntos. Por eso no hace falta un `v2`.

## Impacto en el BFF

- Modelo GraphQL `BloqueDisponibilidad`: `dia`, `horaIni` y `horaFin` pasan a `id`, `ramoCodigo`, `inicio`, `fin` y `libre`.
- Puerto, mock y adaptador gRPC de sesiones: métodos `marcarCasilla`, `desmarcarCasilla` y `buscarDisponibilidad`; rango de fechas en `obtenerDisponibilidad`; filtros en `listar`.
- Resolver:
  - query para la semana del tutor (casillas + reservas en un rango);
  - query para buscar casillas libres por ramo;
  - filtros de estado y fechas en `sesiones`;
  - mutaciones `marcarCasilla` y `desmarcarCasilla`.
- El mock aplica las mismas reglas que Sesiones (bloques válidos, solapes y casillas libres), para que el front pueda trabajar sin el servicio real.

## Pendientes

- Confirmar que no hay tutorías los sábados (la tabla asume lunes a viernes).
- Una vez aprobado, reemplazar `proto/sesiones/v1/sesiones.proto` por el de este documento en un PR, y actualizar Sesiones y el BFF en el mismo momento.
