# Estándares RESTful

## ¿Qué es REST?

REST (**Re**presentational **S**tate **T**ransfer) no es un protocolo ni un estándar — es un **estilo arquitectónico**, un conjunto de restricciones de diseño para sistemas en red. Lo definió Roy Fielding en su tesis doctoral del año 2000. Un sistema que sigue este estilo se llama "RESTful".

Fielding definió 6 restricciones:

1. **Cliente-Servidor:** el frontend y el backend son independientes — se puede cambiar uno sin tocar el otro, mientras se respete el contrato (los formatos de los mensajes).
2. **Stateless (sin estado):** el servidor no guarda memoria de peticiones anteriores del cliente. Cada request tiene que traer toda la información que necesita (ej: un token de autenticación en cada llamada, en vez de que el servidor "recuerde" que ese cliente ya inició sesión). Esto es exactamente el lado Stateless del trade-off [Stateful vs. Stateless](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md) ya visto en punto 1 — REST elige ese lado a propósito, porque favorece el escalado horizontal: cualquier instancia del servidor puede atender cualquier request, sin necesitar la misma instancia que atendió la anterior.
3. **Cacheable:** cada respuesta debe indicar si se puede cachear o no, para que el cliente (o un intermediario) pueda reusar esos datos sin pedirlos de nuevo.
4. **Interfaz Uniforme:** la restricción central de REST, y la que más cuesta cumplir del todo. Se divide en 4 partes:
   - Cada recurso tiene un identificador único (la URI).
   - El cliente manipula el recurso a través de una representación de él (ej: un JSON), no del recurso "real" directamente.
   - Los mensajes son autodescriptivos (el `Content-Type` le dice al receptor cómo procesarlos).
   - **HATEOAS** (Hypermedia as the Engine of Application State): la respuesta debería incluir links a las acciones disponibles desde ahí, para que el cliente descubra qué puede hacer después sin necesitar documentación externa.
5. **Sistema en Capas:** el cliente no puede saber si está hablando directo con el servidor final o con un proxy/gateway en el medio.
6. **Código bajo Demanda (opcional):** la única restricción no obligatoria — el servidor puede mandarle al cliente código ejecutable (ej: JavaScript) para extender su funcionalidad.

**La parte honesta:** casi ninguna API real del mundo implementa el punto 4 completo (HATEOAS). La mayoría de lo que se llama "API REST" en la industria en realidad se queda en un nivel más bajo de un modelo llamado **Richardson Maturity Model** (4 niveles: 0 = un solo endpoint estilo RPC, 1 = recursos con URIs propias, 2 = verbos HTTP correctos + códigos de estado, 3 = HATEOAS). La inmensa mayoría de las APIs "RESTful" del mercado llegan al **Nivel 2**, no al 3 — y eso está perfectamente bien en la práctica: HATEOAS agrega mucha complejidad para un beneficio que casi nadie termina necesitando. Pero vale la pena saber que "usar verbos HTTP y URIs con sustantivos" (que es lo que documenta el resto de este archivo) es REST pragmático, no REST completo según la definición original de Fielding.

## Verbos HTTP

| Verbo | Uso | Idempotente |
|---|---|---|
| `GET` | Leer datos, sin alterar el estado | Sí |
| `POST` | Crear un recurso nuevo | No |
| `PUT` | Reemplazar un recurso completo | Sí |
| `PATCH` | Actualizar parcialmente un recurso (ej: solo el email) | No, en general |
| `DELETE` | Eliminar un recurso | Sí |

**La idempotencia importa en la práctica:** un verbo idempotente significa que llamarlo una vez o cien veces con los mismos datos produce el mismo resultado final. `PUT /users/5` con el mismo body diez veces deja al usuario 5 exactamente igual las diez veces — por eso es seguro reintentarlo automáticamente si la red falla. `POST /users` diez veces con el mismo body, en cambio, podría crear diez usuarios distintos — reintentarlo a ciegas es peligroso.

## Nombres de recursos (URIs semánticas)

Malo (estilo RPC, verbo en la URL):
```
POST /api/obtenerUsuarios
POST /api/actualizarUsuario?id=5
```

Bueno (el recurso es un sustantivo, la acción la define el verbo HTTP):
```
GET /api/v1/users
PATCH /api/v1/users/5
```

**Recursos anidados**, cuando uno pertenece a otro:
```
GET /api/v1/customers/223/orders        → todos los pedidos del cliente 223
GET /api/v1/customers/223/orders/12     → el pedido 12 del cliente 223
```

## Códigos de estado, con sentido semántico

| Código | Cuándo | Nota |
|---|---|---|
| `200 OK` | Operación exitosa con respuesta | |
| `201 Created` | Recurso creado | Debería incluir el header `Location` apuntando al recurso nuevo |
| `204 No Content` | Operación exitosa, sin nada que devolver | Típico de `DELETE` |
| `400 Bad Request` | Error del cliente — sintaxis inválida | Ej: falta un campo obligatorio |
| `401 Unauthorized` | No autenticado | El usuario no probó quién es |
| `403 Forbidden` | Autenticado, pero sin permiso | El usuario ya probó quién es, pero no puede hacer esto |
| `404 Not Found` | El recurso no existe | |
| `422 Unprocessable Entity` | Sintaxis válida, pero viola una regla de negocio | Ej: un cupón con formato correcto pero ya vencido |
| `500 Internal Server Error` | Fallo no controlado del servidor | Un SSR busca minimizar estos al máximo |

La distinción entre 400 y 422 es la que más se confunde: **400 es "no se entendió lo que se mandó"** (falta un campo, un tipo de dato incorrecto). **422 es "se entendió perfecto lo que se mandó, pero no se puede hacer"** (una regla de negocio lo impide). Y 401 vs. 403: **401 es "no se sabe quién es"**, **403 es "ya se sabe quién es, pero no tiene permiso para esto"**.

**Patrón habitual verbo → código** (no es una ley absoluta, pero es lo esperable en la mayoría de los casos):

| Verbo | Código esperado |
|---|---|
| `GET` | `200 OK` |
| `POST` | `201 Created` |
| `PUT` | `200 OK` |
| `DELETE` | `204 No Content` |

## Negociación de contenido: `Accept` y `Content-Type`

El cliente le dice al servidor qué formatos puede entender con el header `Accept`, y el servidor confirma qué formato mandó con `Content-Type`. Ambos usan **MIME types**, con la forma `tipo/subtipo`:

```
GET /api/v1/orders/23
Accept: application/json
```

```
HTTP/1.1 200 OK
Content-Type: application/json
```

MIME types comunes: `application/json` (el más usado en APIs), `application/xml`, `text/html`, `application/pdf`, `image/png`. Si el servidor no puede responder en ninguno de los formatos que el cliente acepta, corresponde `406 Not Acceptable`.

## Estructura estándar de errores (RFC 7807 / RFC 9457)

En vez de inventar un formato de error propio, existe un estándar — RFC 7807 (actualizado por RFC 9457 en 2023, mismo concepto):

```json
{
  "type": "https://example.com/errors/invalid-discount",
  "title": "Regla de negocio violada",
  "status": 400,
  "detail": "El cupón expiró el 01/08/2026."
}
```

ASP.NET Core lo trae integrado — no hace falta armarlo a mano. Con `builder.Services.AddProblemDetails();` en `Program.cs` (que ya está presente en el proyecto real de OrderManagementBackend), las respuestas de error siguen este formato automáticamente.

## Paginación, filtrado y ordenamiento

De las cosas que más se diseñan mal en APIs reales: devolver una lista completa sin ningún límite, hasta que la tabla crece y el endpoint se vuelve lento o directamente falla.

**Paginación por offset** (la más simple, común para listas chicas o medianas):
```
GET /api/v1/orders?page=1&pageSize=20
```

**Paginación por cursor** (mejor para listas grandes o que cambian seguido — evita que un registro nuevo corra el resto de la página):
```
GET /api/v1/orders?cursor=eyJpZCI6MTAwfQ&pageSize=20
```

Filtrado y ordenamiento se agregan como query params adicionales, de forma consistente en toda la API:
```
GET /api/v1/orders?status=Completado&sort=-fecha
```

(el `-` antes de `fecha` es una convención común para indicar orden descendente)
