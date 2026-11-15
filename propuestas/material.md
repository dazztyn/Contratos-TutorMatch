# Servicio de Material: decisiones, modelo de datos y contrato

Propuesta para el equipo de TutorMatch (15-11-2026). Está escrita para que la lea una persona o una IA sin más contexto: resume lo acordado sobre el servicio de Material y Evaluaciones, propone el modelo de MongoDB y el contrato `material.proto` que lo implementa.

Reemplaza al borrador `proto/material/v1/material.proto` del repo de contratos. Ese borrador nunca se implementó, así que se puede reescribir sin romper a nadie. Complementa a `propuestas/usuarios.md`.

## Contexto mínimo

- **TutorMatch:** plataforma donde estudiantes reservan tutorías con otros estudiantes que aprobaron el ramo. Microservicios en Kubernetes.
- **Material** (NestJS + MongoDB, Ien) es dueño del material que suben los tutores (archivos y avisos) y de las evaluaciones que los alumnos hacen de los tutores.
- Cómo se conecta:

| Tramo | Protocolo | Para qué |
|---|---|---|
| BFF → Material | gRPC | Todo lo que pide el front: subir, listar, descargar y eliminar material; evaluar y ver evaluaciones. Los archivos viajan aquí como bytes. |
| Material → Usuarios | gRPC | `ValidarTutorRamo`, antes de aceptar material. |
| Material → Sesiones | gRPC | `VerificarParticipacion`, antes de aceptar una evaluación. |
| Sesiones → Material | Kafka | Eventos `sesion.creada`, `sesion.finalizada` y `sesion.cancelada`, para decidir quién ve qué material. |

- El front nunca llama a Material directamente. Los archivos llegan al BFF por HTTP (`POST /archivos`, multipart) y se descargan de él (`GET /archivos/:id`); el resto va por GraphQL. El BFF verifica el JWT y pone el id del usuario en cada request, así que en este contrato `usuario_id`, `tutor_id` y `estudiante_id` siempre vienen del token, nunca del front.
- Convenciones generales: ids en UUID como texto, fechas en UTC, errores como códigos de estado gRPC.

## Decisiones acordadas

1. **Quién ve el material:**
   - El tutor siempre ve su propio material.
   - Un alumno ve el material de un tutor en un ramo si tiene, o tuvo, al menos una sesión con ese tutor en ese ramo que no esté `CANCELADA`.
   - Material lo decide con su propia copia de las sesiones, armada desde los eventos Kafka. Es lo que le da sentido al tramo Kafka Sesiones → Material, que exige el curso.
2. **Dos tipos de material, en un solo modelo:**
   - `ARCHIVO`: un PDF o una imagen.
   - `AVISO`: solo texto (por ejemplo, "traigan calculadora").

   Ambos se suben, se listan y se eliminan igual, y siguen la misma regla de visibilidad.
3. **Archivos: máximo 10 MB, y solo PDF, PNG y JPEG** (`application/pdf`, `image/png`, `image/jpeg`). Lo validan el BFF y Material. Se guardan en Mongo con GridFS.
4. **Solo el tutor dueño puede eliminar** su material. Eliminar borra el registro y el archivo.
5. **El material se asocia al tutor y al ramo**, no a una sesión específica.
6. **Para subir, el tutor debe tener ese ramo activo** (`Usuarios.ValidarTutorRamo`). Si un tutor quita un ramo, su material de ese ramo sigue visible, pero no puede subir más.
7. **Evaluaciones:**
   - Un alumno evalúa con una nota de 1 a 5 (entero) y un comentario opcional, solo si la sesión está `FINALIZADA` y él participó (`Sesiones.VerificarParticipacion`).
   - Una evaluación por sesión.
   - No se editan ni se borran.
8. **Las evaluaciones son anónimas:**
   - Material guarda quién la escribió (lo necesita para validar y para impedir duplicados), pero **nunca** devuelve `estudiante_id` ni `sesion_id` a nadie más que al propio alumno. Con el id de la sesión se sabría quién la escribió, porque cada sesión tiene un solo alumno.
   - El anonimato es parcial: si un tutor tuvo una sola sesión en una fecha, puede deducir de quién es. Para este proyecto basta con ocultar los ids.
9. **Las evaluaciones son públicas para los usuarios con sesión iniciada:** cualquier alumno ve el promedio, la cantidad y los comentarios de un tutor (sin autor), y el tutor ve lo mismo de sí mismo. El buscador de tutores muestra el promedio con un RPC en lote.
10. **Sin quizzes por ahora:** el agente todavía no está definido, así que Material no extrae texto de los PDF. Si se necesita después, se agrega como campo nuevo.

## Flujos del front

| Caso | Llamada en el BFF | RPC de Material |
|---|---|---|
| Tutor sube un PDF o una imagen | `POST /archivos` (multipart) | `SubirMaterial` con `archivo` |
| Tutor publica un aviso | mutación GraphQL | `SubirMaterial` con `texto` |
| Tutor ve su material, por ramo | query GraphQL | `ListarMaterial` |
| Alumno ve el material de un tutor | query GraphQL | `ListarMaterial` (solo devuelve lo que puede ver) |
| Alumno descarga o abre un archivo | `GET /archivos/:id` | `DescargarArchivo` |
| Tutor elimina material | mutación GraphQL | `EliminarMaterial` |
| Alumno evalúa una sesión finalizada | mutación GraphQL | `Evaluar` |
| Alumno ve qué sesiones ya evaluó | query GraphQL | `ListarMisEvaluaciones` |
| Ver las evaluaciones de un tutor | query GraphQL | `ListarEvaluaciones` |
| Buscador de tutores con su promedio | query GraphQL | `ObtenerResumenTutores` (en lote) |

## Modelo de MongoDB

Base de datos `material`. Los ids son UUID en texto, guardados en `_id`.

### `materiales`

```js
// Archivo
{
  _id: "6b0d…", tutorId: "a1b2…", ramoCodigo: "MAT1610",
  tipo: "ARCHIVO", titulo: "Guía de integrales",
  archivo: { gridfsId: ObjectId("…"), nombre: "guia.pdf", tipoMime: "application/pdf", tamanoBytes: 482113 },
  creadoEn: ISODate("2026-11-20T18:00:00Z")
}
// Aviso
{
  _id: "9c4e…", tutorId: "a1b2…", ramoCodigo: "MAT1610",
  tipo: "AVISO", titulo: "Para el jueves", texto: "Traigan calculadora.",
  creadoEn: ISODate("2026-11-20T18:05:00Z")
}
```

Los bytes de los archivos van en el bucket de GridFS `archivos` (colecciones `archivos.files` y `archivos.chunks`), no en el documento: un documento de Mongo no puede pasar de 16 MB.

### `sesiones` (copia armada desde Kafka)

```js
{ _id: "3f2a…", tutorId: "a1b2…", estudianteId: "c3d4…", ramoCodigo: "MAT1610", estado: "CANCELADA" }
```

`estado` es `CREADA`, `FINALIZADA` o `CANCELADA`. Confirmar una sesión no publica evento, así que una confirmada sigue como `CREADA`; para la regla de visibilidad da lo mismo.

### `evaluaciones`

```js
{
  _id: "7e1f…", sesionId: "3f2a…", tutorId: "a1b2…", estudianteId: "c3d4…",
  ramoCodigo: "MAT1610", nota: 5, comentario: "Explica muy bien.",
  creadaEn: ISODate("2026-11-21T20:00:00Z")
}
```

### Índices

```js
db.materiales.createIndex({ tutorId: 1, ramoCodigo: 1, creadoEn: -1 });
db.sesiones.createIndex({ estudianteId: 1, tutorId: 1, ramoCodigo: 1 });
// Una evaluación por sesión: cada sesión tiene un solo alumno.
db.evaluaciones.createIndex({ sesionId: 1 }, { unique: true });
db.evaluaciones.createIndex({ tutorId: 1, creadaEn: -1 });
db.evaluaciones.createIndex({ estudianteId: 1 });
```

### Consumidor de Kafka

- Grupo de consumidores `material`, suscrito a `sesion.creada`, `sesion.finalizada` y `sesion.cancelada`.
- La primera vez lee los tópicos desde el inicio (`fromBeginning`), para no perder sesiones creadas antes de que Material existiera.
- Formato de los mensajes: `eventos/sesion.*.json` en este repo. La clave del mensaje es el `sesionId`, así que los eventos de una sesión llegan en orden.
- Cada evento se aplica como un *upsert* que da el mismo resultado si llega repetido. Por eso no hace falta guardar los `eventId` procesados:

```js
// sesion.creada: solo crea; si la sesión ya existe (evento repetido), no la toca.
db.sesiones.updateOne(
  { _id: p.sesionId },
  { $setOnInsert: { tutorId: p.tutorId, estudianteId: p.estudianteId, ramoCodigo: p.ramoCodigo, estado: "CREADA" } },
  { upsert: true });

// sesion.finalizada y sesion.cancelada: fijan el estado (y crean la sesión si el evento de creación se perdió).
db.sesiones.updateOne(
  { _id: p.sesionId },
  { $set: { tutorId: p.tutorId, estudianteId: p.estudianteId, ramoCodigo: p.ramoCodigo, estado: "CANCELADA" } }, // o "FINALIZADA"
  { upsert: true });
```

### Regla de visibilidad (decisión 1)

```js
// ¿Puede usuarioId ver el material de tutorId en ramoCodigo?
usuarioId === tutorId ||
  db.sesiones.countDocuments(
    { estudianteId: usuarioId, tutorId, ramoCodigo, estado: { $ne: "CANCELADA" } },
    { limit: 1 }) > 0
```

`ListarMaterial` aplica esta regla a cada par tutor-ramo y omite lo que el usuario no puede ver.

### Resumen de un tutor (decisión 9)

```js
db.evaluaciones.aggregate([
  { $match: { tutorId: { $in: tutorIds } } },
  { $group: { _id: "$tutorId", promedio: { $avg: "$nota" }, cantidad: { $sum: 1 } } }
]);
```

Los tutores sin evaluaciones no aparecen en el resultado; `ObtenerResumenTutores` los devuelve con promedio 0 y cantidad 0.

### Límite de tamaño en gRPC

gRPC corta los mensajes en 4 MB por defecto. Con archivos de hasta 10 MB, hay que subir el límite a **11 MB** (10 MB más margen para los demás campos) en los dos extremos:

- Material, en el servidor: `maxReceiveMessageLength` y `maxSendMessageLength`.
- BFF, en el cliente de Material: los mismos dos.

El BFF también limita el tamaño en `POST /archivos`, para rechazar un archivo grande antes de reenviarlo.

## Contrato `material.proto`

```proto
// Contrato gRPC del servicio de Material y Evaluaciones.
// Lo implementa Material (NestJS + MongoDB, Ien) y lo consume el BFF.
//
// El material pertenece a un tutor y a un ramo. Hay dos tipos: ARCHIVO (PDF, PNG o JPEG,
// hasta 10 MB) y AVISO (solo texto).
//
// Quién ve el material:
//   - El tutor siempre ve su propio material.
//   - Un alumno ve el material de un tutor en un ramo si tiene o tuvo al menos una sesión
//     con ese tutor en ese ramo que no esté CANCELADA.
//   Material lo decide con su propia copia de las sesiones, armada desde los eventos
//   Kafka sesion.creada, sesion.finalizada y sesion.cancelada (ver eventos/).
//
// Las evaluaciones son anónimas: Material guarda quién la escribió, pero solo se lo
// devuelve al propio alumno (ListarMisEvaluaciones). Nunca expone estudiante_id ni
// sesion_id en las vistas públicas.
//
// usuario_id, tutor_id y estudiante_id los pone el BFF a partir del JWT.
//
// Dependencias de Material:
//   - Usuarios.ValidarTutorRamo antes de aceptar un SubirMaterial.
//   - Sesiones.VerificarParticipacion antes de aceptar una evaluación.
//
// Los mensajes llevan archivos de hasta 10 MB: el servidor y sus clientes configuran un
// límite de 11 MB por mensaje (gRPC usa 4 MB por defecto).
//
// Los errores viajan como códigos de estado gRPC, no como campos de la respuesta:
//   NOT_FOUND            el material no existe o el usuario no puede verlo; la sesión no existe.
//   ALREADY_EXISTS       esa sesión ya fue evaluada.
//   PERMISSION_DENIED    quien sube no tiene ese ramo activo como tutor, o quien elimina
//                        no es el dueño del material.
//   FAILED_PRECONDITION  la sesión no está FINALIZADA o el alumno no participó.
//   INVALID_ARGUMENT     faltan datos; el archivo supera 10 MB o no es PDF, PNG ni JPEG;
//                        el aviso no tiene texto; la nota está fuera de 1..5; el comentario
//                        supera 1000 caracteres; se pidió descargar un aviso.
//   UNAVAILABLE          Usuarios o Sesiones no responden.
syntax = "proto3";

package tutorias.material.v1;

import "google/protobuf/timestamp.proto";

service MaterialService {
  rpc SubirMaterial(SubirMaterialRequest) returns (Material);
  // Devuelve solo el material que usuario_id puede ver, del más nuevo al más antiguo;
  // lo demás se omite, sin error. Nunca incluye los bytes de los archivos.
  rpc ListarMaterial(ListarMaterialRequest) returns (ListarMaterialResponse);
  // Bytes de un archivo, con la misma regla de visibilidad que ListarMaterial.
  rpc DescargarArchivo(DescargarArchivoRequest) returns (DescargarArchivoResponse);
  // Solo el dueño. Borra el registro y el archivo.
  rpc EliminarMaterial(EliminarMaterialRequest) returns (EliminarMaterialResponse);

  rpc Evaluar(EvaluarRequest) returns (EvaluacionPropia);
  // Evaluaciones de un tutor, anónimas. Las ve cualquier usuario con sesión iniciada.
  rpc ListarEvaluaciones(ListarEvaluacionesRequest) returns (ListarEvaluacionesResponse);
  // Evaluaciones escritas por un alumno, para saber qué sesiones ya evaluó.
  rpc ListarMisEvaluaciones(ListarMisEvaluacionesRequest) returns (ListarMisEvaluacionesResponse);
  // En lote, para mostrar el promedio de muchos tutores en el buscador con una sola llamada.
  rpc ObtenerResumenTutores(ObtenerResumenTutoresRequest) returns (ObtenerResumenTutoresResponse);
}

enum TipoMaterial {
  // proto3 exige un valor 0; nunca se guarda ni se devuelve.
  TIPO_MATERIAL_DESCONOCIDO = 0;
  ARCHIVO = 1;
  AVISO = 2;
}

// Datos de un archivo guardado, sin los bytes.
message Archivo {
  string nombre = 1;
  // "application/pdf", "image/png" o "image/jpeg".
  string tipo_mime = 2;
  int64 tamano_bytes = 3;
}

message Material {
  string id = 1;
  // Dueño del material.
  string tutor_id = 2;
  string ramo_codigo = 3;
  TipoMaterial tipo = 4;
  string titulo = 5;
  // Solo en AVISO.
  string texto = 6;
  // Solo en ARCHIVO.
  Archivo archivo = 7;
  google.protobuf.Timestamp creado_en = 8;
}

// Archivo tal como lo sube el tutor.
message ArchivoSubido {
  string nombre = 1;
  string tipo_mime = 2;
  // Hasta 10 MB.
  bytes datos = 3;
}

message SubirMaterialRequest {
  string tutor_id = 1;
  string ramo_codigo = 2;
  string titulo = 3;
  // Exactamente uno: texto crea un AVISO, archivo crea un ARCHIVO.
  oneof contenido {
    string texto = 4;
    ArchivoSubido archivo = 5;
  }
}

// tutor_id y ramo_codigo son filtros opcionales: un campo vacío no filtra.
message ListarMaterialRequest {
  // Quién consulta. Obligatorio: define qué material es visible.
  string usuario_id = 1;
  string tutor_id = 2;
  string ramo_codigo = 3;
}

message ListarMaterialResponse {
  repeated Material materiales = 1;
}

message DescargarArchivoRequest {
  string id = 1;
  string usuario_id = 2;
}

message DescargarArchivoResponse {
  Archivo archivo = 1;
  bytes datos = 2;
}

message EliminarMaterialRequest {
  string id = 1;
  string tutor_id = 2;
}

message EliminarMaterialResponse {}

message EvaluarRequest {
  string sesion_id = 1;
  string estudiante_id = 2;
  // Entero de 1 a 5.
  int32 nota = 3;
  // Opcional, hasta 1000 caracteres.
  string comentario = 4;
}

// Vista pública de una evaluación: sin quién la escribió ni de qué sesión es.
message EvaluacionAnonima {
  string ramo_codigo = 1;
  int32 nota = 2;
  string comentario = 3;
  google.protobuf.Timestamp creada_en = 4;
}

// Una evaluación vista por el alumno que la escribió.
message EvaluacionPropia {
  string sesion_id = 1;
  string tutor_id = 2;
  string ramo_codigo = 3;
  int32 nota = 4;
  string comentario = 5;
  google.protobuf.Timestamp creada_en = 6;
}

message ListarEvaluacionesRequest {
  string tutor_id = 1;
}

message ListarEvaluacionesResponse {
  // De la más nueva a la más antigua.
  repeated EvaluacionAnonima evaluaciones = 1;
  // 0 si el tutor no tiene evaluaciones.
  double promedio = 2;
  int32 cantidad = 3;
}

message ListarMisEvaluacionesRequest {
  string estudiante_id = 1;
}

message ListarMisEvaluacionesResponse {
  repeated EvaluacionPropia evaluaciones = 1;
}

message ResumenTutor {
  string tutor_id = 1;
  // 0 si el tutor no tiene evaluaciones.
  double promedio = 2;
  int32 cantidad = 3;
}

message ObtenerResumenTutoresRequest {
  repeated string tutor_ids = 1;
}

message ObtenerResumenTutoresResponse {
  // Uno por cada id pedido, incluidos los que no tienen evaluaciones.
  repeated ResumenTutor resumenes = 1;
}
```

## Cambio relacionado en `sesiones.proto`

Para guardar una evaluación, Material necesita el tutor y el ramo de la sesión. Hoy `VerificarParticipacion` solo responde `finalizada` y `participo`. Se propone agregar dos campos con números nuevos, un cambio seguro que no rompe a nadie:

```proto
message VerificarParticipacionResponse {
  bool finalizada = 1;
  bool participo = 2;
  string tutor_id = 3;
  string ramo_codigo = 4;
}
```

Así `Evaluar` hace una sola llamada a Sesiones, que es la fuente de verdad, y no depende de que la copia de Kafka esté al día. Lo implementa Sesiones (Vicente).

## Qué cambia respecto al borrador del repo de contratos

| Borrador | Esta propuesta | Motivo |
|---|---|---|
| Material de texto (`contenido`) | `tipo` `ARCHIVO` o `AVISO`; los archivos van como bytes y se guardan en GridFS | PDF, imágenes y avisos. |
| `sesion_id` opcional en el material | Se quita | El material se asocia al tutor y al ramo. |
| — | `DescargarArchivo`, `EliminarMaterial` | Descargar y eliminar archivos. |
| `Evaluacion` con `estudiante_id` y `sesion_id` | `EvaluacionAnonima` (pública) y `EvaluacionPropia` (solo el autor) | Evaluaciones anónimas. |
| `ListarEvaluaciones` con `promedio` | Agrega `cantidad` | Mostrar "4,6 (12 evaluaciones)". |
| — | `ListarMisEvaluaciones` | El alumno sabe qué sesiones ya evaluó. |
| — | `ObtenerResumenTutores` | Promedio en el buscador, en lote. |
| Contenido pensado para el quiz del agente | Sin quiz por ahora | El agente todavía no está definido. |
| Dependencia `Usuarios.VerificarTutorRamo` | `Usuarios.ValidarTutorRamo` | Nombre nuevo, según `propuestas/usuarios.md`. |

## Pendientes

- Que Ien revise el modelo de Mongo contra lo que ya tenga pensado.
- Aprobar el cambio de `VerificarParticipacionResponse` en `sesiones.proto`.
- Confirmar el largo máximo del comentario (por ahora, 1000 caracteres).
- Una vez aprobado, reemplazar `proto/material/v1/material.proto` en el repo de contratos por PR y avisar al BFF y a Sesiones.
