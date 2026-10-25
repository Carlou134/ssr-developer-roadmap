# Estándares RESTful

## En una frase

Una API REST es **un contrato predecible**: los recursos son sustantivos en la URL, el verbo HTTP dice qué se hace, el código de estado dice qué pasó, y los errores, la paginación y las versiones siguen siempre el mismo formato, para que nadie tenga que leer tu código para saber cómo usarla.

-----

## Antes de empezar

Conviene que ya sepas:

* Lo básico de HTTP: petición, respuesta, headers y body.
* El trade-off stateful frente a stateless: [¿Qué son los trade-offs?](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md).
* Por qué no exponer entidades: [Arquitectura en capas](../02-calidad-de-codigo-y-arquitectura/08-arquitectura-en-capas-ntier.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **REST (*Representational State Transfer*):** estilo arquitectónico para sistemas en red, definido por un conjunto de restricciones.
* **Recurso:** cualquier cosa que la API expone y que tiene una identidad, como un pedido o un cliente.
* **URI:** el identificador de un recurso, como `/api/v1/orders/12`.
* **Representación:** el formato en que viaja un recurso, normalmente JSON.
* **Idempotente:** que ejecutar la operación una vez o muchas veces deja el mismo estado final.
* **Método seguro:** método HTTP que no modifica el estado del servidor, como `GET`.
* **HATEOAS:** que la respuesta incluya links a las acciones disponibles desde ahí.
* **Problem Details:** formato estándar para describir errores en APIs HTTP (RFC 9457).
* **MIME type (*media type*):** identificador del formato de un contenido, como `application/json`.
* **Paginación:** devolver una lista en partes (páginas) en lugar de completa.

-----

## El problema

Una API creció sin convenciones:

```text
POST /api/obtenerPedidos
POST /api/pedido/crear
GET  /api/eliminarPedido?id=12
POST /api/Clientes/GetById
```

```http
POST /api/pedido/crear
→ 200 OK
{ "success": false, "error": "El cupón expiró" }
```

```http
POST /api/obtenerPedidos
→ 200 OK
[ ... 48 000 pedidos ... ]
```

Las consecuencias:

* el frontend no puede saber si algo falló mirando el código de estado: tiene que leer `success` en cada respuesta;
* `GET /api/eliminarPedido` borra datos, y un rastreador web o un prefetch del navegador puede dispararlo;
* el listado de pedidos tarda 9 segundos y crece cada día;
* cada endpoint nuevo inventa su propio formato de error, así que cada cliente tiene que manejar cinco formatos distintos.

-----

## Cómo funciona

### 1. Qué es REST (y qué no)

REST no es un protocolo ni un estándar: es un **estilo arquitectónico**, un conjunto de restricciones de diseño. Lo definió Roy Fielding en su tesis doctoral del año 2000. Un sistema que sigue ese estilo se llama *RESTful*.

| # | Restricción | Qué significa |
| --- | --- | --- |
| 1 | **Cliente-servidor** | Frontend y backend son independientes: se puede cambiar uno sin tocar el otro mientras se respete el contrato |
| 2 | **Sin estado (*stateless*)** | El servidor no recuerda peticiones anteriores: cada petición trae todo lo que necesita (por ejemplo, el token en cada llamada) |
| 3 | **Cacheable** | Cada respuesta indica si se puede guardar en caché y por cuánto tiempo |
| 4 | **Interfaz uniforme** | La restricción central (ver abajo) |
| 5 | **Sistema en capas** | El cliente no sabe si habla con el servidor final o con un proxy o gateway |
| 6 | **Código bajo demanda** (opcional) | El servidor puede enviar código ejecutable al cliente, como JavaScript |

La restricción 2 es el lado *stateless* del trade-off de la [fase 1](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md). REST lo elige a propósito porque favorece el escalado horizontal: cualquier instancia puede atender cualquier petición.

La **interfaz uniforme** se divide en cuatro partes:

* cada recurso tiene un identificador único (la URI);
* el cliente manipula el recurso a través de una **representación** (un JSON), no del recurso "real";
* los mensajes son autodescriptivos (el `Content-Type` dice cómo procesarlos);
* **HATEOAS**: la respuesta incluye links a las acciones disponibles, para que el cliente descubra qué puede hacer sin documentación externa.

**La parte honesta:** casi ninguna API real implementa HATEOAS. El **modelo de madurez de Richardson** lo muestra:

| Nivel | Qué tiene | Ejemplo |
| --- | --- | --- |
| 0 | Un solo endpoint, estilo RPC | `POST /api` con la acción en el body |
| 1 | Recursos con URIs propias | `/orders/12`, pero todo con `POST` |
| 2 | Verbos HTTP correctos y códigos de estado | `GET /orders/12` → `200`, `DELETE` → `204` |
| 3 | HATEOAS | La respuesta incluye `"links": [{ "rel": "cancel", "href": "..." }]` |

La inmensa mayoría de las APIs "RESTful" del mercado llegan al **nivel 2**, y está bien: HATEOAS agrega mucha complejidad para un beneficio que casi nadie necesita. Pero conviene saber que lo que documenta esta lección es **REST pragmático**, no REST completo según Fielding.

### 2. Verbos HTTP

| Verbo | Uso | Seguro | Idempotente |
| --- | --- | --- | --- |
| `GET` | Leer, sin alterar el estado | Sí | Sí |
| `POST` | Crear un recurso nuevo (o una acción que no encaja en otro verbo) | No | No |
| `PUT` | Reemplazar un recurso completo | No | Sí |
| `PATCH` | Actualizar una parte del recurso (por ejemplo, solo el email) | No | No, en general |
| `DELETE` | Eliminar un recurso | No | Sí |

**Seguro** significa que no modifica el estado. **Idempotente** significa que llamarlo una vez o cien veces con los mismos datos produce el mismo estado final.

En la práctica, la idempotencia decide si es seguro **reintentar**:

* `PUT /users/5` con el mismo body diez veces deja al usuario 5 igual las diez veces: se puede reintentar automáticamente si falla la red.
* `POST /users` diez veces con el mismo body puede crear diez usuarios: reintentarlo a ciegas es peligroso.
* `DELETE /orders/12` dos veces: la primera borra, la segunda devuelve `404`. La **respuesta** cambia, pero el **estado** final es el mismo (el pedido no existe), así que sigue siendo idempotente.

Si lo usas mal:

```http
GET /api/orders/12/delete
```

`GET` es seguro por definición. Navegadores, rastreadores, proxies y herramientas de *prefetch* asumen que pueden llamarlo sin consecuencias. Un `GET` que borra datos es un incidente esperando a ocurrir.

### 3. Nombres de recursos

| Malo (estilo RPC: verbo en la URL) | Bueno (el recurso es un sustantivo, el verbo lo da HTTP) |
| --- | --- |
| `POST /api/obtenerUsuarios` | `GET /api/v1/users` |
| `POST /api/actualizarUsuario?id=5` | `PATCH /api/v1/users/5` |
| `GET /api/eliminarPedido?id=12` | `DELETE /api/v1/orders/12` |

Convenciones habituales:

* sustantivos en **plural** (`/orders`, no `/order`);
* minúsculas y guiones para nombres compuestos (`/order-items`);
* sin verbos ni extensiones (`/users`, no `/getUsers` ni `/users.json`).

**Recursos anidados**, cuando uno pertenece a otro:

```text
GET /api/v1/customers/223/orders        → todos los pedidos del cliente 223
GET /api/v1/customers/223/orders/12     → el pedido 12 del cliente 223
```

Evita anidar más de dos niveles (`/customers/1/orders/2/items/3/discounts`): se vuelve frágil. Si el recurso tiene identidad propia, dale su propia URI (`/order-items/3`).

**Acciones que no son CRUD.** Algunas operaciones no encajan en crear, leer, actualizar o borrar, como "cancelar un pedido". Hay dos opciones aceptadas:

```text
POST /api/v1/orders/12/cancellation     → se crea una "cancelación" (un sub-recurso)
POST /api/v1/orders/12/cancel           → una acción explícita, pragmática y muy común
```

### 4. Códigos de estado con sentido semántico

| Código | Cuándo | Nota |
| --- | --- | --- |
| `200 OK` | Operación exitosa con respuesta | |
| `201 Created` | Recurso creado | Debería incluir el header `Location` con la URI del recurso nuevo |
| `204 No Content` | Operación exitosa, sin nada que devolver | Típico de `DELETE` |
| `400 Bad Request` | La petición está mal formada | Falta un campo obligatorio, un tipo de dato incorrecto |
| `401 Unauthorized` | No autenticado | No se sabe quién es |
| `403 Forbidden` | Autenticado, pero sin permiso | Se sabe quién es, pero no puede hacer esto |
| `404 Not Found` | El recurso no existe | |
| `409 Conflict` | Choca con el estado actual del recurso | Un email que ya está registrado, una versión desactualizada |
| `422 Unprocessable Content` | La petición se entendió, pero viola una regla de negocio | Un cupón con formato correcto pero vencido |
| `429 Too Many Requests` | Se superó el límite de peticiones | Suele ir con el header `Retry-After` |
| `500 Internal Server Error` | Fallo no controlado del servidor | Un SSR busca minimizarlos |

Las dos distinciones que más se confunden:

* **400 frente a 422:** 400 es "no se entendió lo que se mandó"; 422 es "se entendió perfecto, pero no se puede hacer".
* **401 frente a 403:** 401 es "no sé quién eres"; 403 es "sé quién eres, pero no tienes permiso".

> **Ojo con el nombre del 422:** en RFC 4918 se llamaba "Unprocessable Entity". RFC 9110 (2022) lo incorporó a HTTP como **"Unprocessable Content"**. En .NET la constante sigue siendo `StatusCodes.Status422UnprocessableEntity`.

Patrón habitual de verbo a código (lo esperable, no una ley):

| Verbo | Éxito |
| --- | --- |
| `GET` | `200 OK` |
| `POST` | `201 Created` (con `Location`) |
| `PUT` / `PATCH` | `200 OK` con el recurso, o `204 No Content` |
| `DELETE` | `204 No Content` |

Si lo usas mal:

```http
HTTP/1.1 200 OK
{ "success": false, "error": "Pedido no encontrado" }
```

Devolver `200` con un error en el body rompe a los clientes, proxies, monitores y reintentos automáticos, que miran el código de estado para decidir qué hacer. Tu herramienta de monitoreo mostrará 0 % de errores mientras la API falla.

### 5. Negociación de contenido: `Accept` y `Content-Type`

El cliente dice qué formatos entiende con `Accept`, y el servidor indica qué formato envió con `Content-Type`. Los dos usan **MIME types** (`tipo/subtipo`):

```http
GET /api/v1/orders/23
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

MIME types comunes: `application/json`, `application/xml`, `text/html`, `application/pdf`, `image/png`, y para errores, `application/problem+json`.

Si el servidor no puede responder en ninguno de los formatos que acepta el cliente, corresponde `406 Not Acceptable`. Si el cliente envía un body en un formato que el servidor no soporta, `415 Unsupported Media Type`.

### 6. Estructura estándar de errores: Problem Details

En lugar de inventar un formato de error propio, existe un estándar: **RFC 9457** (2023), que reemplazó a RFC 7807 (2016) con el mismo concepto.

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.tienda.com/errors/cupon-vencido",
  "title": "El cupón no es válido",
  "status": 422,
  "detail": "El cupón VERANO25 expiró el 01/08/2026.",
  "instance": "/api/v1/orders"
}
```

| Campo | Qué es |
| --- | --- |
| `type` | URI que identifica el tipo de error; los clientes la usan para decidir qué hacer |
| `title` | Resumen corto y estable del tipo de error |
| `status` | El mismo código HTTP de la respuesta |
| `detail` | Explicación de **esta** ocurrencia |
| `instance` | URI de la petición concreta que falló |

Se pueden agregar campos propios (por ejemplo, `"errors"` con los errores de validación por campo, o un `"traceId"`).

ASP.NET Core lo trae integrado, pero no todo es automático:

```csharp
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();    // excepciones no controladas → Problem Details
app.UseStatusCodePages();     // 404, 405, etc. sin body → Problem Details
```

* Los errores de validación del modelo en controllers con `[ApiController]` ya devuelven `ValidationProblemDetails` (un `400` con el campo `errors`), incluso sin `AddProblemDetails()`.
* Sin `UseExceptionHandler()`, una excepción no controlada **no** se convierte en Problem Details.
* Para traducir tus excepciones de negocio a un `422` o `409`, implementa un `IExceptionHandler` (.NET 8).

### 7. Paginación, filtrado y ordenamiento

Es de lo que más se diseña mal: devolver una lista completa sin límite, hasta que la tabla crece y el endpoint se vuelve lento o falla.

| | Offset | Cursor |
| --- | --- | --- |
| Ejemplo | `?page=3&pageSize=20` | `?cursor=eyJpZCI6MTAwfQ&pageSize=20` |
| Ventaja | Simple; permite saltar a la página N | Rápido en tablas grandes; estable si se insertan registros |
| Desventaja | Lento en páginas lejanas (la base recorre todo lo anterior); si se inserta un registro, la página "corre" y se repite o se pierde uno | No permite saltar a una página arbitraria |
| Ideal para | Listas chicas o medianas, tablas con número de página | Feeds, scroll infinito, listas grandes que cambian seguido |

El cursor suele ser el último valor visto codificado en Base64 (`{"id":100}` → `eyJpZCI6MTAwfQ`), y la consulta pide "los siguientes después de ese".

Filtrado y ordenamiento como query params, **con los mismos nombres en toda la API**:

```text
GET /api/v1/orders?status=Completado&sort=-fecha&page=1&pageSize=20
```

(El `-` antes de `fecha` es una convención común para el orden descendente).

La respuesta debería incluir los metadatos de paginación:

```json
{
  "items": [ ... ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 482
}
```

> **Ojo:** pon siempre un `pageSize` máximo en el servidor (por ejemplo, 100). Si no, un cliente puede pedir `?pageSize=1000000` y volver al problema original.

### 8. Versionado

El `v1` de las URLs no es decoración: cuando un cambio **rompe** el contrato (quitar un campo, cambiar su tipo, cambiar el significado de un código), los clientes existentes dejan de funcionar. Versionar permite publicar el cambio sin romperlos.

| Estrategia | Ejemplo | Comentario |
| --- | --- | --- |
| En la URL | `/api/v1/orders` | La más común y la más visible |
| En un query param | `/api/orders?api-version=2.0` | Habitual en APIs de Microsoft |
| En un header | `Api-Version: 2` | URLs más limpias, menos visible |

Qué **no** rompe el contrato (no necesita versión nueva): agregar un endpoint, agregar un campo opcional a la respuesta, agregar un parámetro opcional. Qué **sí** lo rompe: quitar o renombrar campos, cambiar tipos, hacer obligatorio un parámetro que era opcional.

En .NET, el paquete `Asp.Versioning.Mvc` implementa las tres estrategias.

-----

## Ejemplo completo

La API del problema, rediseñada.

| Antes | Después |
| --- | --- |
| `POST /api/obtenerPedidos` | `GET /api/v1/orders?status=&sort=&page=&pageSize=` |
| `POST /api/pedido/crear` | `POST /api/v1/orders` → `201` + `Location` |
| `GET /api/eliminarPedido?id=12` | `DELETE /api/v1/orders/12` → `204` |
| `POST /api/Clientes/GetById` | `GET /api/v1/customers/{id}` → `200` o `404` |
| `200` + `"success": false` | `422` + Problem Details |

`OrdersController.cs`:

```csharp
[ApiController]
[Route("api/v1/orders")]
public class OrdersController(OrderService service) : ControllerBase
{
    private const int PageSizeMaximo = 100;

    [HttpGet]
    public async Task<ActionResult<PagedResult<OrderSummaryDto>>> List(
        [FromQuery] string? status, [FromQuery] string? sort,
        [FromQuery] int page = 1, [FromQuery] int pageSize = 20, CancellationToken ct = default)
    {
        pageSize = Math.Clamp(pageSize, 1, PageSizeMaximo);
        return await service.ListAsync(status, sort, page, pageSize, ct);
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<OrderDto>> Get(int id, CancellationToken ct) =>
        await service.GetAsync(id, ct) is { } dto ? dto : NotFound();

    [HttpPost]
    public async Task<ActionResult<OrderDto>> Create(CreateOrderRequest request, CancellationToken ct)
    {
        var dto = await service.CreateAsync(request, ct);
        return CreatedAtAction(nameof(Get), new { id = dto.Id }, dto);
    }

    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id, CancellationToken ct)
    {
        await service.DeleteAsync(id, ct);
        return NoContent();
    }
}
```

`ExcepcionesDeNegocioHandler.cs` (las reglas de negocio → `422` con Problem Details):

```csharp
public class ExcepcionesDeNegocioHandler(IProblemDetailsService problemDetails) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext http, Exception ex, CancellationToken ct)
    {
        (int status, string title, string type) = ex switch
        {
            CuponVencidoException => (422, "El cupón no es válido", "cupon-vencido"),
            NotFoundException => (404, "Recurso no encontrado", "no-encontrado"),
            _ => (0, "", "")
        };
        if (status == 0) return false; // que lo maneje el handler por defecto (500)

        http.Response.StatusCode = status;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = http,
            ProblemDetails = new ProblemDetails
            {
                Status = status,
                Title = title,
                Type = $"https://api.tienda.com/errors/{type}",
                Detail = ex.Message
            }
        });
    }
}
```

`Program.cs`:

```csharp
builder.Services.AddControllers();
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<ExcepcionesDeNegocioHandler>();

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages();
app.MapControllers();
app.Run();
```

Resultado de crear un pedido con un cupón vencido:

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.tienda.com/errors/cupon-vencido",
  "title": "El cupón no es válido",
  "status": 422,
  "detail": "El cupón VERANO25 expiró el 01/08/2026."
}
```

Qué observar:

* Ningún verbo en las URLs: la acción la da el método HTTP.
* `DELETE` ya no se puede disparar con un `GET` accidental.
* Un solo formato de error para toda la API, y el código de estado dice la verdad.
* La lista nunca devuelve más de 100 elementos, pida lo que pida el cliente.
* `CreatedAtAction` arma el `201` con el header `Location: /api/v1/orders/{id}`.

-----

## Errores comunes

**1. Verbos en las URLs.**
Qué pasa: `/api/getUsers`, `/api/createOrder`: la API se vuelve un catálogo de funciones remotas.
Por qué: se piensa en métodos del código, no en recursos.
Arreglo: sustantivos en plural y el verbo HTTP para la acción.

**2. `200 OK` para todo.**
Qué pasa: los clientes, los monitores y los reintentos no detectan los errores.
Por qué: se pone el resultado en el body (`"success": false`).
Arreglo: el código de estado correcto y Problem Details en el body.

**3. `GET` que modifica datos.**
Qué pasa: datos borrados o modificados por un rastreador, un prefetch o un reintento automático.
Por qué: se usa `GET` "porque es más fácil de probar en el navegador".
Arreglo: `GET` solo lee; para modificar, `POST`, `PUT`, `PATCH` o `DELETE`.

**4. Listas sin paginar.**
Qué pasa: un endpoint que tarda cada vez más hasta que falla por timeout o por memoria.
Por qué: con pocos datos en desarrollo, nadie lo nota.
Arreglo: paginación desde el primer día, con un `pageSize` máximo en el servidor.

**5. Confundir 401 y 403.**
Qué pasa: el frontend redirige al login a un usuario que ya inició sesión pero no tiene permiso.
Por qué: se usa 401 para "no puede hacer esto".
Arreglo: 401 si no está autenticado; 403 si está autenticado y no tiene permiso.

**6. Cambios que rompen el contrato sin versión nueva.**
Qué pasa: una app móvil en producción deja de funcionar tras un despliegue del backend.
Por qué: se renombró o quitó un campo "porque nadie lo usaba".
Arreglo: los cambios incompatibles van en una versión nueva; los compatibles (agregar campos opcionales) no la necesitan.

-----

## Evolución y versiones

* **2000:** Roy Fielding define REST en su tesis doctoral.
* **2008:** Leonard Richardson presenta el modelo de madurez de cuatro niveles.
* **2016:** RFC 7807 define Problem Details.
* **2022:** RFC 9110 consolida la semántica de HTTP e incorpora `422 Unprocessable Content`.
* **2023:** RFC 9457 reemplaza a RFC 7807.
* **.NET 7 y 8:** `AddProblemDetails()`, `IProblemDetailsService` e `IExceptionHandler` integran Problem Details en ASP.NET Core.

-----

## Cuándo sí y cuándo no

**Usa REST cuando:** expones recursos bien definidos (pedidos, clientes, productos) a clientes web, móviles u otros servicios, y quieres aprovechar el caché HTTP y la infraestructura estándar.

**Considera otra opción cuando:** los clientes necesitan combinaciones muy variables de datos (ver [GraphQL](02-graphql.md)), la comunicación es entre servicios internos con alto rendimiento (gRPC) o el modelo es de eventos (colas, WebSockets).

-----

## Resumen en 5 líneas

1. REST es un estilo con restricciones; casi todas las APIs llegan al nivel 2 de Richardson (recursos + verbos + códigos), y está bien.
2. Recursos como sustantivos en plural; la acción la da el verbo HTTP.
3. La idempotencia decide si es seguro reintentar; `GET` nunca modifica datos.
4. El código de estado dice la verdad (400/422, 401/403), y los errores van con Problem Details (RFC 9457).
5. Toda lista se pagina con un máximo en el servidor, y los cambios que rompen el contrato van en una versión nueva.

-----

## Para profundizar

<details>
<summary>Idempotency keys: hacer seguro reintentar un <code>POST</code></summary>

`POST` no es idempotente, pero a veces hay que reintentarlo (un pago que dio timeout: ¿se cobró o no?). La solución que usan APIs como Stripe es la **clave de idempotencia**: el cliente genera un identificador único y lo envía en un header.

```http
POST /api/v1/payments
Idempotency-Key: 7f3c1b2e-9a4d-4e1f-8c6b-2d5a9e0f1a3b
```

El servidor guarda la respuesta asociada a esa clave. Si llega otra petición con la misma clave, devuelve la respuesta guardada en lugar de cobrar otra vez. Así el cliente puede reintentar sin miedo a duplicar.

</details>

<details>
<summary>Caché HTTP: <code>ETag</code> y peticiones condicionales</summary>

El servidor puede enviar un `ETag` (una huella de la versión del recurso):

```http
HTTP/1.1 200 OK
ETag: "a1b2c3"
Cache-Control: max-age=60
```

En la siguiente petición, el cliente envía `If-None-Match: "a1b2c3"`. Si el recurso no cambió, el servidor responde `304 Not Modified` sin body, y se ahorra el ancho de banda.

El mismo `ETag` sirve para evitar que dos usuarios se pisen los cambios: con `If-Match: "a1b2c3"` en un `PUT`, si el recurso cambió mientras tanto, el servidor responde `412 Precondition Failed` (control de concurrencia optimista).

</details>

-----

## En entrevista

### Respuesta corta (junior)

En una API REST los recursos se nombran con sustantivos (`/users`) y la acción la indica el verbo HTTP: `GET` lee, `POST` crea, `PUT` reemplaza, `PATCH` actualiza una parte y `DELETE` borra. Se usan los códigos de estado correctos, como `201` al crear o `404` si no existe.

### Respuesta ampliada (semi-senior)

Diseño la API como un contrato predecible: recursos en plural, verbos HTTP con su semántica (sabiendo cuáles son seguros e idempotentes, porque eso decide qué se puede reintentar), y códigos de estado que dicen la verdad, distinguiendo 400 de 422 y 401 de 403. Todos los errores salen en formato Problem Details (RFC 9457), que en ASP.NET Core configuro con `AddProblemDetails`, `UseExceptionHandler` y un `IExceptionHandler` para las excepciones de negocio. Pagino siempre las listas, por offset o por cursor según el tamaño y la volatilidad, con un máximo en el servidor. Y versiono cuando un cambio rompe el contrato. Sé que eso es REST pragmático, nivel 2 de Richardson: HATEOAS casi nunca vale su costo.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `PUT` y `PATCH`?**
`PUT` reemplaza el recurso completo y es idempotente. `PATCH` modifica solo una parte y, en general, no lo es.

**2. ¿`DELETE` es idempotente si la segunda llamada devuelve `404`?**
Sí. La idempotencia se refiere al estado final del servidor, no a la respuesta: después de una o de diez llamadas, el recurso no existe.

**3. ¿Cuándo usarías paginación por cursor?**
En listas grandes o que cambian seguido, como un feed, porque el offset se vuelve lento en páginas lejanas y "corre" los resultados cuando se insertan registros.

**4. ¿Qué es HATEOAS y por qué casi nadie lo usa?**
Que las respuestas incluyan links a las acciones posibles. Casi nadie lo usa porque los clientes reales se programan contra una documentación, no descubriendo links en tiempo de ejecución, y el costo de mantenerlo no se recupera.

-----

## Práctica

**Ejercicio 1.** Rediseña estos endpoints con convenciones REST e indica el código de éxito de cada uno.

```text
POST /api/listarProductos
POST /api/producto/nuevo
POST /api/producto/editarPrecio?id=7
GET  /api/producto/borrar?id=7
POST /api/obtenerResenasDeProducto?id=7
```

<details>
<summary>Solución</summary>

| Antes | Después | Éxito |
| --- | --- | --- |
| `POST /api/listarProductos` | `GET /api/v1/products?page=1&pageSize=20` | `200` |
| `POST /api/producto/nuevo` | `POST /api/v1/products` | `201` + `Location` |
| `POST /api/producto/editarPrecio?id=7` | `PATCH /api/v1/products/7` con `{ "price": 99.9 }` | `200` o `204` |
| `GET /api/producto/borrar?id=7` | `DELETE /api/v1/products/7` | `204` |
| `POST /api/obtenerResenasDeProducto?id=7` | `GET /api/v1/products/7/reviews` | `200` |

</details>

**Ejercicio 2.** ¿Qué código de estado devolverías en cada caso?

1. El body es un JSON mal formado.
2. El usuario no envió token.
3. El usuario es "lector" e intenta borrar un pedido.
4. Se intenta registrar un email que ya existe.
5. Se intenta cancelar un pedido que ya fue enviado.
6. El cliente superó las 100 peticiones por minuto.

<details>
<summary>Solución</summary>

1. **400 Bad Request:** no se pudo entender la petición.
2. **401 Unauthorized:** no se sabe quién es.
3. **403 Forbidden:** se sabe quién es, pero no tiene permiso.
4. **409 Conflict:** choca con el estado actual (el email ya existe).
5. **422 Unprocessable Content:** la petición es válida, pero una regla de negocio lo impide. Hay equipos que usan 409 para este caso; lo importante es ser consistente en toda la API.
6. **429 Too Many Requests**, idealmente con `Retry-After`.

</details>

-----

## Siguiente lección

[GraphQL](02-graphql.md)
