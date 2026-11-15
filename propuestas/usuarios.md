# Servicio de Usuarios: decisiones, esquema y contrato

Propuesta para el equipo de TutorMatch (15-11-2026). Está escrita para que la lea una persona o una IA sin más contexto: resume lo acordado sobre el servicio de Usuarios, propone el esquema de PostgreSQL y el contrato `usuarios.proto` que lo implementa.

Reemplaza al borrador `proto/usuarios/v1/usuarios.proto` del repo de contratos. Ese borrador nunca se implementó, así que se puede reescribir sin romper a nadie. Usa los nombres de RPC que propuso Pablo, que es quien implementa el servicio.

## Contexto mínimo

- **TutorMatch:** plataforma donde estudiantes reservan tutorías con otros estudiantes que aprobaron el ramo. Microservicios en Kubernetes.
- **Usuarios** (Go + PostgreSQL, Pablo) es dueño de las cuentas, el login, el catálogo de ramos y qué ramos imparte cada tutor.
- Lo llaman por gRPC:
  - **BFF** (NestJS + GraphQL, Vicente): registro, login, datos de usuarios, ramos y búsqueda de tutores.
  - **Sesiones** (Spring Boot, Vicente): antes de crear una reserva o marcar disponibilidad, valida que el tutor imparte ese ramo.
  - **Material** (NestJS, Ien): antes de aceptar material, valida lo mismo.
  - **Agente** (MCP, Pablo): busca tutores por ramo.
- Convenciones generales: ids en UUID como texto, fechas en UTC, errores como códigos de estado gRPC.

## Decisiones acordadas

1. **Un mismo usuario puede ser alumno y tutor a la vez**, como en las ayudantías. No hay roles: es tutor de un ramo cuando se inscribió en él.
2. **Un solo login y dos portales en el front**, como en un campus virtual. "Ingreso de alumnos" e "Ingreso de tutores" usan el mismo RPC `Login`, con el mismo email y contraseña. El front decide qué pantalla abrir según `es_tutor`.
3. **`es_tutor` es una columna guardada**, con tres reglas para que nunca contradiga a los ramos:
   1. Nadie lo escribe directamente: no hay RPC ni campo de request para cambiarlo. En el contrato solo se lee.
   2. Se recalcula en un solo lugar, siempre con la misma fórmula: es verdadero si el usuario tiene al menos un ramo **activo**. Lo recalculan las funciones que cambian ramos (`AgregarRamosTutor`, `QuitarRamoTutor`, `ReactivarRamoTutor`).
   3. Se recalcula en la **misma transacción** que el cambio de ramos: si algo falla, el `ROLLBACK` deshace las dos cosas.
4. **Inscribirse como tutor es un formulario de verificación simulado.** Si alguien sin ramos entra por el portal de tutores, se le invita a inscribirse. El formulario pide, por cada ramo, la nota con que lo aprobó y el periodo. Si los datos son válidos, la inscripción se aprueba en el acto (es un proyecto académico, no hay revisión manual).
5. **Un tutor puede quitar un ramo y volver a activarlo.** Quitar no borra la inscripción: la marca inactiva y conserva los datos de verificación. Reactivar la vuelve a activar sin repetir el formulario. Si un tutor queda sin ramos activos, `es_tutor` pasa a `false`.
6. **Los permisos no dependen de `es_tutor`.** `es_tutor` solo decide qué pantalla ve el usuario. Lo que protege las acciones de tutor es `ValidarTutorRamo`, que Sesiones y Material llaman con el ramo concreto. Por eso el JWT no lleva rol.
7. **Quitar un ramo no cancela nada.** Las sesiones ya reservadas de ese ramo siguen su curso (el tutor puede cancelarlas en Sesiones). Lo que se impide son reservas nuevas, porque `ValidarTutorRamo` responde `false` para un ramo inactivo.

## Flujos del front

| Caso | Llamadas |
|---|---|
| Registro de alumno | `RegistroUsuario`. Devuelve el token, así que queda con la sesión iniciada. |
| Registro de tutor | `RegistroUsuario` y luego el formulario: `AgregarRamosTutor`. Si el formulario falla, la cuenta queda como alumno y puede reintentarlo desde el portal de tutores. |
| Ingreso por el portal de alumnos | `Login` → pantalla de alumno, sea o no tutor. |
| Ingreso por el portal de tutores | `Login` → si `es_tutor`, pantalla de tutor; si no, invitación al formulario de inscripción (`AgregarRamosTutor`). |
| Tutor deja de impartir un ramo | `QuitarRamoTutor`. |
| Tutor vuelve a impartirlo | `ReactivarRamoTutor`. |
| Selector de ramos (registro, filtros) | `ListarRamos`. |
| Alumno busca tutores de un ramo | `ListarTutoresPorRamo`. |

El front nunca llama a Usuarios directamente: todo pasa por el BFF por GraphQL. El BFF verifica el JWT y saca de él el id del usuario, así que en los RPC que modifican la cuenta el `usuario_id` lo pone el BFF, nunca el front.

## Esquema de PostgreSQL

```sql
-- Catálogo de ramos. Es la referencia de ramo_codigo para Sesiones y Material.
CREATE TABLE ramos (
    codigo TEXT PRIMARY KEY,          -- ej. 'MAT1610'
    nombre TEXT NOT NULL              -- ej. 'Cálculo II'
);

CREATE TABLE usuarios (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre        TEXT        NOT NULL,
    email         TEXT        NOT NULL,
    password_hash TEXT        NOT NULL,  -- bcrypt; nunca se devuelve por gRPC
    -- Pendiente: datos personales que pida el formulario de registro (ej. carrera).
    -- Derivado de tutor_ramos; solo lo escribe la función que recalcula (regla 3).
    es_tutor      BOOLEAN     NOT NULL DEFAULT false,
    creado_en     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- El email es único sin importar mayúsculas.
CREATE UNIQUE INDEX usuarios_email_unico ON usuarios (lower(email));

-- Inscripción de un usuario como tutor de un ramo, con los datos del formulario.
CREATE TABLE tutor_ramos (
    usuario_id         UUID         NOT NULL REFERENCES usuarios (id) ON DELETE CASCADE,
    ramo_codigo        TEXT         NOT NULL REFERENCES ramos (codigo),
    -- Escala chilena: aprobado es de 4.0 a 7.0.
    nota_aprobacion    NUMERIC(2,1) NOT NULL CHECK (nota_aprobacion BETWEEN 4.0 AND 7.0),
    -- Año y semestre, ej. '2025-2'.
    periodo_aprobacion TEXT         NOT NULL CHECK (periodo_aprobacion ~ '^[0-9]{4}-[12]$'),
    -- false = el tutor lo quitó; se conserva para poder reactivarlo sin repetir el formulario.
    activo             BOOLEAN      NOT NULL DEFAULT true,
    verificado_en      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    PRIMARY KEY (usuario_id, ramo_codigo)
);

-- Para ListarTutoresPorRamo y ValidarTutorRamo.
CREATE INDEX tutor_ramos_activos_por_ramo ON tutor_ramos (ramo_codigo) WHERE activo;
```

### Recalcular `es_tutor` (reglas 2 y 3)

Una sola sentencia, siempre la misma, que se ejecuta al final de cada función que cambia ramos y **dentro de su transacción**:

```sql
UPDATE usuarios
SET es_tutor = EXISTS (
    SELECT 1 FROM tutor_ramos WHERE usuario_id = $1 AND activo
)
WHERE id = $1;
```

Como calcula el valor a partir de los ramos en vez de poner `true` o `false` a mano, da el resultado correcto sin importar qué cambió antes.

Ejemplo en Go, para `QuitarRamoTutor`:

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
    return err
}
defer tx.Rollback() // si no se llega al Commit, deshace todo; después del Commit no hace nada

res, err := tx.ExecContext(ctx,
    `UPDATE tutor_ramos SET activo = false WHERE usuario_id = $1 AND ramo_codigo = $2`,
    usuarioID, ramoCodigo)
if err != nil {
    return err
}
if n, _ := res.RowsAffected(); n == 0 {
    return status.Error(codes.NotFound, "el usuario no está inscrito en ese ramo")
}
if err := recalcularEsTutor(ctx, tx, usuarioID); err != nil { // el UPDATE de arriba
    return err
}
return tx.Commit()
```

Importante: todas las sentencias van por `tx`, no por `db`. Si una va por `db`, Postgres la confirma por su cuenta y el rollback ya no la deshace.

## Contrato `usuarios.proto`

```proto
// Contrato gRPC del servicio de Usuarios.
// Lo implementa Usuarios (Go, Pablo) y lo consumen el BFF, Sesiones, Material y el agente.
//
// Un mismo usuario puede ser alumno y tutor a la vez. Es tutor de un ramo cuando se
// inscribió en él con el formulario de verificación (AgregarRamosTutor) y no lo ha quitado.
// es_tutor es verdadero si tiene al menos un ramo activo; solo se lee, nunca se escribe.
//
// Los RPC que modifican la cuenta reciben usuario_id del BFF, que lo saca del JWT.
//
// Los errores viajan como códigos de estado gRPC, no como campos de la respuesta:
//   NOT_FOUND          el usuario no existe, el ramo no está en el catálogo, o el usuario
//                      no está inscrito en el ramo que quiere quitar o reactivar.
//   ALREADY_EXISTS     ya hay un usuario con ese email, o ya está inscrito en ese ramo
//                      (si lo quitó, se usa ReactivarRamoTutor).
//   UNAUTHENTICATED    email o contraseña incorrectos.
//   INVALID_ARGUMENT   faltan datos o tienen mal formato, la nota está fuera de 4.0..7.0
//                      o el periodo no tiene la forma "2025-2".
syntax = "proto3";

package tutorias.usuarios.v1;

option go_package = "github.com/tutormatch/usuarios/gen/usuariosv1";

service UsuariosService {
  // Crea la cuenta y deja la sesión iniciada: responde igual que Login.
  rpc RegistroUsuario(RegistroUsuarioRequest) returns (LoginResponse);
  // Devuelve un JWT firmado (HS256) con el id del usuario en "sub" y el vencimiento en
  // "exp" (24 horas). No lleva rol. El BFF lo verifica con el mismo secreto (variable
  // JWT_SECRET, Secret de Kubernetes) y rechaza los tokens vencidos.
  rpc Login(LoginRequest) returns (LoginResponse);

  rpc ObtenerUsuario(ObtenerUsuarioRequest) returns (Usuario);
  // En lote, para mostrar el tutor y el alumno de muchas sesiones con una sola llamada.
  // Los ids que no existen se omiten de la respuesta, sin error.
  rpc ObtenerUsuarios(ObtenerUsuariosRequest) returns (ObtenerUsuariosResponse);

  // Formulario de verificación: inscribe al usuario como tutor de uno o más ramos.
  // Todo o nada: si un ramo falla, no se inscribe ninguno.
  rpc AgregarRamosTutor(AgregarRamosTutorRequest) returns (Usuario);
  // Deja de impartir un ramo. Conserva la inscripción para poder reactivarla.
  // Quitar un ramo ya inactivo no hace nada y no es error.
  rpc QuitarRamoTutor(CambiarRamoTutorRequest) returns (Usuario);
  // Vuelve a impartir un ramo que había quitado, sin repetir el formulario.
  // Reactivar un ramo ya activo no hace nada y no es error.
  rpc ReactivarRamoTutor(CambiarRamoTutorRequest) returns (Usuario);

  // Catálogo de ramos, para los selectores del front.
  rpc ListarRamos(ListarRamosRequest) returns (ListarRamosResponse);
  // Tutores con ese ramo activo. Lo usan el buscador de alumnos y el agente.
  rpc ListarTutoresPorRamo(ListarTutoresPorRamoRequest) returns (ListarTutoresPorRamoResponse);

  // Usado por Sesiones (al crear una reserva y al marcar disponibilidad) y por Material
  // (al subir material). valido es true solo si el usuario existe y tiene ese ramo activo.
  rpc ValidarTutorRamo(ValidarTutorRamoRequest) returns (ValidarTutorRamoResponse);
}

message Ramo {
  // Ej. "MAT1610". Es el mismo ramo_codigo que usan Sesiones y Material.
  string codigo = 1;
  string nombre = 2;
}

// Un ramo en el que el usuario se inscribió como tutor.
message RamoTutor {
  Ramo ramo = 1;
  // false si el tutor lo quitó; se muestra para que pueda reactivarlo.
  bool activo = 2;
}

message Usuario {
  // UUID. Es el tutor_id / estudiante_id que guardan Sesiones y Material.
  string id = 1;
  string nombre = 2;
  string email = 3;
  // Verdadero si tiene al menos un ramo activo. El front lo usa para elegir el portal.
  bool es_tutor = 4;
  repeated RamoTutor ramos = 5;
  // Pendiente: datos personales que pida el formulario de registro, con números nuevos.
}

message RegistroUsuarioRequest {
  string nombre = 1;
  string email = 2;
  string password = 3;
  // Pendiente: datos personales que pida el formulario de registro, con números nuevos.
}

message LoginRequest {
  string email = 1;
  string password = 2;
}

message LoginResponse {
  string token = 1;
  Usuario usuario = 2;
}

message ObtenerUsuarioRequest {
  string id = 1;
}

message ObtenerUsuariosRequest {
  repeated string ids = 1;
}

message ObtenerUsuariosResponse {
  repeated Usuario usuarios = 1;
}

// Datos del formulario de verificación para un ramo.
message InscripcionRamo {
  string ramo_codigo = 1;
  // Escala chilena, de 4.0 a 7.0.
  double nota_aprobacion = 2;
  // Año y semestre, ej. "2025-2".
  string periodo_aprobacion = 3;
}

message AgregarRamosTutorRequest {
  string usuario_id = 1;
  repeated InscripcionRamo ramos = 2;
}

message CambiarRamoTutorRequest {
  string usuario_id = 1;
  string ramo_codigo = 2;
}

message ListarRamosRequest {}

message ListarRamosResponse {
  repeated Ramo ramos = 1;
}

message ListarTutoresPorRamoRequest {
  string ramo_codigo = 1;
}

message ListarTutoresPorRamoResponse {
  repeated Usuario tutores = 1;
}

message ValidarTutorRamoRequest {
  string tutor_id = 1;
  string ramo_codigo = 2;
}

message ValidarTutorRamoResponse {
  bool valido = 1;
}
```

## Qué cambia respecto al borrador del repo de contratos

| Borrador | Esta propuesta | Motivo |
|---|---|---|
| `Registrar` → `Usuario` | `RegistroUsuario` → token + usuario | Nombre de Pablo; queda con la sesión iniciada sin otra llamada. |
| `IniciarSesion` | `Login` | Nombre de Pablo. Agrega `exp` (24 h) al JWT. |
| `ActualizarRamos` (reemplaza la lista) | `AgregarRamosTutor`, `QuitarRamoTutor`, `ReactivarRamoTutor` | Formulario de verificación y poder quitar y reactivar sin repetirlo. |
| `BuscarTutores` | `ListarTutoresPorRamo` | Nombre de Pablo. Solo ramos activos. |
| `VerificarTutorRamo` (`existe`, `aprobo_ramo`) | `ValidarTutorRamo` (`valido`) | Nombre de Pablo. A Sesiones y Material les basta un booleano. |
| `Usuario.ramos_aprobados` | `Usuario.es_tutor` + `Usuario.ramos` con `activo` | Portales del front y ramos inactivos. |
| `ObtenerUsuarios`, `ListarRamos` | Se mantienen | El BFF los necesita (nombres en lote y catálogo de ramos). |

## Pendientes

- Definir los datos personales del formulario de registro y agregarlos a `usuarios`, `Usuario` y `RegistroUsuarioRequest`.
- Confirmar los campos del formulario de verificación (por ahora: nota y periodo de aprobación).
- Definir quién carga el catálogo de `ramos` (por ejemplo, una migración con los ramos de la carrera).
- Una vez aprobado, reemplazar `proto/usuarios/v1/usuarios.proto` en el repo de contratos por PR y avisar al BFF, Sesiones, Material y el agente.
