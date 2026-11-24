# Contrato REST del agente (MCP server)

Lo implementa el agente (`tutormatch-agente`, Go, Pablo) y lo consume el BFF (`AGENTE_URL`, por defecto `http://localhost:8081` en local y `http://agente:8081` en Kubernetes).

## `POST /agent/chat`

El BFF valida el JWT, envía el `usuarioId` en el cuerpo y **reenvía la cabecera `Authorization: Bearer <token>`** para que el agente la propague al llamar al BFF por GraphQL (`BFF_URL`). De esta forma, el BFF sigue siendo quien determina la identidad del estudiante al crear una reserva.

Cabeceras:
- `Content-Type: application/json`
- `Authorization: Bearer <token>`

Petición:
```json
{
  "usuarioId": "a1b2c3d4-0000-4000-8000-000000000002",
  "conversacionId": "conv-123",
  "mensaje": "Necesito ayuda con Cálculo II el jueves en la tarde"
}
```

`conversacionId` es opcional (máx. 128 caracteres): si falta, el agente abre una conversación nueva y devuelve su id. Las conversaciones se guardan indexadas por usuario y conversación (`usuarioId + conversacionId`), por lo que un usuario no puede acceder al historial de otro.

Respuesta `200`:
```json
{
  "conversacionId": "conv-123",
  "respuesta": "Encontré opciones disponibles para Cálculo II...",
  "acciones": [
    { "tipo": "sesion.creada", "id": "3f2a9c1e-5b7d-4e8a-9c21-7d4e5f6a8b90" }
  ]
}
```

`acciones` es una lista vacía `[]` si el agente solo conversó, o incluye `{ "tipo": "sesion.creada", "id": "<sesionId>" }` cuando creó una reserva para que el frontend refresque las vistas.

Errores:
- `400`: falta `usuarioId` o `mensaje`, el mensaje excede 2000 caracteres o `conversacionId` excede 128 caracteres.
- `429`: límite de solicitudes por usuario excedido.
- `503`: el modelo de IA no responde.
- `504`: el agente superó el máximo de pasos permitidos.

## Tools MCP (`POST /mcp`)

Las tools consumen el esquema GraphQL real del BFF (`BFF_URL`) usando bloques universitarios de 90 minutos (A–F, hora de Chile `America/Santiago`, enviados en UTC):

| Tool | Operación en el BFF |
|---|---|
| `listar_ramos()` | `Query.ramos` |
| `buscar_tutores(ramo_codigo)` | `Query.tutoresPorRamo` |
| `buscar_disponibilidad(ramo_codigo, desde, hasta)` | `Query.buscarDisponibilidad` |
| `ver_disponibilidad(tutor_id, desde, hasta)` | `Query.disponibilidad` |
| `crear_reserva(tutor_id, ramo_codigo, inicio)` | `Mutation.crearSesion` (exige `Authorization`) |

`generar_quiz` queda postergada mientras Material no extraiga texto de los archivos.

## Restricciones de operación en Kubernetes

- **Réplica única (`replicas: 1`):** El historial de conversaciones vive en memoria (caché LRU con expiración TTL) y se pierde si el pod se reinicia. El Deployment debe correr con una sola réplica para evitar que los turnos de una misma conversación caigan en pods distintos (escalar horizontalmente requeriría mover el store a Redis).
- **Red interna (`ClusterIP`):** El Service `agente:8081` **no debe exponerse fuera del clúster**, porque `/mcp` acepta `X-Usuario-Id` directamente y está pensado solo para uso interno.
