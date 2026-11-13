# Contrato REST del agente (MCP server)

BORRADOR para revisar en equipo. Lo implementa el agente (Pablo) y lo consume el BFF.

## `POST /agent/chat`

El BFF ya verificó el token, así que envía el id del usuario en el cuerpo. El agente actúa siempre en nombre de ese usuario (por ejemplo, al crear una reserva, `usuarioId` es el estudiante).

Petición:

```json
{
  "usuarioId": "a1b2c3d4-0000-4000-8000-000000000002",
  "conversacionId": "c-123",
  "mensaje": "Necesito ayuda con Cálculo II el jueves en la tarde"
}
```

`conversacionId` es opcional: si falta, el agente abre una conversación nueva y devuelve su id.

Respuesta `200`:

```json
{
  "conversacionId": "c-123",
  "respuesta": "Encontré 2 tutores con MAT1610 disponibles el jueves de 15:00 a 18:00...",
  "acciones": [
    { "tipo": "sesion.creada", "id": "3f2a9c1e-5b7d-4e8a-9c21-7d4e5f6a8b90" }
  ]
}
```

`acciones` lista lo que el agente hizo en el sistema, para que el front refresque esas vistas. Vacío si solo conversó.

Errores: `400` si falta `usuarioId` o `mensaje`; `503` si el modelo de IA no responde.

## Tools que expone el MCP server

| Tool | Llama a |
|---|---|
| `buscar_tutores(ramoCodigo)` | `Usuarios.BuscarTutores` |
| `ver_disponibilidad(tutorId)` | `Sesiones.ObtenerDisponibilidad` |
| `crear_reserva(tutorId, ramoCodigo, inicio, fin)` | `Sesiones.CrearSesion` |
| `generar_quiz(ramoCodigo)` | `Material.ListarMaterial` + el modelo |

Las tools llaman a los servicios por gRPC directo con los `.proto` de este repo.
