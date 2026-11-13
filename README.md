# tutormatch-contracts

Contratos compartidos de TutorMatch. No se despliega: cada servicio lo incluye como submódulo de git en la carpeta `contracts/`.

```bash
git submodule add <url-de-este-repo> contracts
# al clonar un servicio:
git clone --recurse-submodules <url-del-servicio>
```

En GitHub Actions, el checkout necesita `submodules: true`.

## Contenido

| Archivo | Lo implementa | Lo consumen |
|---|---|---|
| `proto/sesiones/v1/sesiones.proto` | Sesiones (Vicente) | BFF, Material, agente |
| `proto/usuarios/v1/usuarios.proto` | Usuarios (Pablo) | BFF, Sesiones, agente |
| `proto/material/v1/material.proto` | Material (Ien) | BFF, agente |
| `eventos/sesion.*.json` | Sesiones publica | Material consume |
| `rest/agente.md` | Agente (Pablo) | BFF |

## Kafka

- Un tópico por tipo de evento, con el mismo nombre que su `type`: `sesion.creada`, `sesion.finalizada`, `sesion.cancelada`.
- La clave del mensaje es el `sesionId`, así los eventos de una sesión llegan en orden.
- Todos los mensajes usan el mismo sobre: `eventId`, `type`, `occurredAt` (ISO-8601 UTC) y `payload`.
- Un evento puede llegar repetido: el consumidor ignora un `eventId` que ya procesó.
- Un evento también puede perderse si Kafka no responde. Por eso Material confirma con `Sesiones.VerificarParticipacion` antes de aceptar una evaluación.

## Convenciones

- Ids de usuario y de sesión: UUID en texto.
- Códigos de ramo: los del catálogo de `Usuarios.ListarRamos` (ej. `MAT1610`).
- Errores: códigos de estado gRPC, documentados al inicio de cada `.proto`.
- Direcciones en Kubernetes: `<servicio>:9090` para gRPC (`sesiones:9090`, `usuarios:9090`, `material:9090`), leídas de `SESIONES_GRPC_URL`, `USUARIOS_GRPC_URL` y `MATERIAL_GRPC_URL`.

## Reglas para cambiar un contrato

1. Proponer el cambio en un PR y avisar a quienes lo consumen.
2. Agregar campos con números nuevos es seguro.
3. Nunca reutilizar ni cambiar el número de un campo. Al eliminar uno, marcarlo con `reserved`.
4. Cambiar tipos o renombrar RPC rompe a los clientes: si es inevitable, crear `v2`.
