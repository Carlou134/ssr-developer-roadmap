# Documentación técnica

## En una frase

Un Semi-Senior mantiene **tres tipos de documentación con propósitos distintos**: la especificación OpenAPI para saber **cómo** usar la API, los contratos (DTOs) acordados antes de programar para que frontend y backend no se pisen, y los ADRs para saber **por qué** se tomó cada decisión.

-----

## Antes de empezar

Conviene que ya sepas:

* Verbos, códigos de estado y Problem Details: [Estándares RESTful](01-estandares-restful.md).
* Qué es un DTO: [Arquitectura en capas](../02-calidad-de-codigo-y-arquitectura/08-arquitectura-en-capas-ntier.md).
* Qué es un ADR: [¿Qué son los trade-offs?](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **OpenAPI:** especificación estándar para describir APIs HTTP en un archivo JSON o YAML.
* **Swagger:** el nombre original de OpenAPI (hasta 2015) y hoy el de un conjunto de herramientas, como Swagger UI.
* **Swashbuckle:** librería de .NET que genera la especificación OpenAPI y sirve Swagger UI.
* **Contrato:** el acuerdo sobre qué recibe y qué devuelve una API, en qué formato y con qué errores.
* **Contract-first / code-first:** escribir primero el contrato y generar el código, o escribir el código y generar el contrato.
* **Breaking change:** cambio en el contrato que rompe a los clientes existentes.
* **ADR (*Architecture Decision Record*):** documento corto que registra una decisión técnica, sus alternativas y por qué se eligió.

-----

## El problema

El equipo de frontend empieza a integrar la API de pedidos. La documentación de Swagger muestra esto para `POST /api/v1/orders`:

```text
POST /api/v1/orders
Responses:
  200  Success
```

Durante la integración descubren, uno por uno:

* que en realidad devuelve `201`, no `200`;
* que puede devolver `400` con un formato y `422` con otro;
* que el campo `customerId` es obligatorio, aunque Swagger no lo marca;
* que a mitad del sprint el backend renombró `total` a `totalAmount`, y la pantalla de confirmación dejó de mostrar el monto.

Seis meses después, alguien nuevo pregunta por qué los pedidos usan PostgreSQL y los productos usan MongoDB. Nadie lo recuerda.

Las consecuencias:

* la documentación generada no refleja lo que la API hace de verdad;
* el contrato se fue definiendo mientras se programaba, y los dos equipos reescribieron código;
* el porqué de una decisión importante se perdió.

-----

## Cómo funciona

### 1. Los tres tipos de documentación

| Tipo | Responde | Quién la usa | Herramienta |
| --- | --- | --- | --- |
| **Documentación de API** | ¿**Cómo** uso esta API? | Quien la consume: frontend, móvil, otros equipos | OpenAPI + Swagger UI o Scalar |
| **Contratos (DTOs)** | ¿**Qué** datos se intercambian? | Frontend y backend, antes de programar | DTOs, especificación OpenAPI acordada |
| **Decisiones (ADRs)** | ¿**Por qué** está hecho así? | El equipo, hoy y dentro de un año | Archivos Markdown en el repositorio |

Swagger resuelve el primero, pero no los otros dos.

### 2. OpenAPI y Swagger

**OpenAPI** es la especificación: un archivo JSON o YAML que describe cada endpoint, sus parámetros, sus cuerpos de petición y respuesta, y sus códigos de estado. **Swagger** fue su nombre original hasta 2015; hoy el nombre se usa para las herramientas, como **Swagger UI**, la página web que permite leer y probar la API.

En .NET hay dos formas de generar la especificación:

| | Swashbuckle | `Microsoft.AspNetCore.OpenApi` |
| --- | --- | --- |
| Qué es | Librería de la comunidad | Paquete oficial de Microsoft |
| En la plantilla de .NET | Hasta .NET 8 | Desde .NET 9 |
| UI incluida | Swagger UI | Ninguna: se agrega Swagger UI o Scalar aparte |

```csharp
// .NET 9 o posterior
builder.Services.AddOpenApi();

var app = builder.Build();
app.MapOpenApi();                 // expone /openapi/v1.json
app.MapScalarApiReference();      // UI interactiva (paquete Scalar.AspNetCore)
```

```csharp
// Hasta .NET 8, con Swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();
app.UseSwagger();
app.UseSwaggerUI();
```

> **Ojo:** si sigues un tutorial de .NET 8 en un proyecto de .NET 9, no vas a encontrar `AddSwaggerGen()` en la plantilla. No es que esté roto: Microsoft lo reemplazó por su propio generador. Puedes agregar Swashbuckle a mano si lo prefieres.

### 3. Documentar todos los códigos, no solo el 200

Si el método devuelve `IActionResult`, el generador no puede saber qué respuestas existen y muestra un `200` genérico. Hay que declararlas:

```csharp
[HttpPost]
[ProducesResponseType<OrderDto>(StatusCodes.Status201Created)]
[ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
[ProducesResponseType<ProblemDetails>(StatusCodes.Status422UnprocessableEntity)]
public async Task<IActionResult> CrearPedido(CrearPedidoDto dto)
{
    // ...
}
```

Sin esto, quien consume la API solo ve "200 OK" en la documentación, y descubre los demás casos en producción.

Alternativa más segura: **resultados tipados** (desde .NET 7), donde el tipo de retorno ya dice qué puede devolver el método, y el compilador te obliga a cumplirlo:

```csharp
[HttpPost]
public async Task<Results<Created<OrderDto>, ValidationProblem, UnprocessableEntity<ProblemDetails>>>
    CrearPedido(CrearPedidoDto dto)
{
    // ...
}
```

Con `ProducesResponseType`, la documentación puede mentir si alguien agrega un `return NotFound()` y olvida el atributo. Con resultados tipados, ese `return` no compila hasta que agregas `NotFound` al tipo.

### 4. Comentarios XML: descripciones legibles

Los comentarios `///` de C# pueden aparecer en la documentación, pero solo si el proyecto genera el archivo de documentación:

```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn> <!-- no exigir comentarios en todo lo público -->
</PropertyGroup>
```

```csharp
/// <summary>Crea un pedido para un cliente existente.</summary>
/// <remarks>El total se calcula en el servidor; no se acepta desde el cliente.</remarks>
/// <response code="201">El pedido se creó.</response>
/// <response code="422">El cupón no es válido o el stock no alcanza.</response>
[HttpPost]
public async Task<IActionResult> CrearPedido(CrearPedidoDto dto) { /* ... */ }
```

Con `Microsoft.AspNetCore.OpenApi`, estos comentarios se incorporan a la especificación automáticamente desde .NET 10; en .NET 9 todavía no. Con Swashbuckle hay que indicarle la ruta del archivo XML con `IncludeXmlComments`.

### 5. Contratos (DTOs) definidos antes de programar

Los DTOs entre frontend y backend se definen **antes** de empezar a programar, no mientras tanto. Un DTO que cambia a mitad de camino obliga a los dos equipos a reescribir código en paralelo.

| | Code-first | Contract-first |
| --- | --- | --- |
| Orden | Se escribe el código y se genera la especificación | Se escribe la especificación OpenAPI y se genera o valida el código |
| Ventaja | Rápido; la especificación nunca se desincroniza del código | Frontend y backend trabajan en paralelo desde el día 1 |
| Desventaja | El contrato aparece recién cuando el backend está hecho | Hay que mantener la especificación y validar que el código la cumpla |

Un punto intermedio muy práctico: acordar los DTOs (o un borrador de la especificación) en una reunión corta al inicio de la funcionalidad, versionarlos en el repositorio, y que el frontend arranque con un servidor de prueba (*mock*) generado desde esa especificación.

El DTO también protege el dominio: es la **superficie pública** del contrato, mientras que la entidad es un detalle interno que puede cambiar sin romper a nadie afuera.

> **Ojo con los breaking changes:** renombrar `total` a `totalAmount` es un cambio que rompe el contrato, aunque "el dato sea el mismo". Agregar un campo nuevo no rompe; renombrar, quitar o cambiar el tipo, sí. Ver [Versionado](01-estandares-restful.md).

### 6. Decisiones de arquitectura (ADRs)

El ADR ya se vio en detalle en [¿Qué son los trade-offs?](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md). Un ADR corto que explica por qué se tomó una decisión (por ejemplo, "se eligió PostgreSQL sobre MongoDB por la necesidad de transacciones ACID en pagos") es tan parte de la documentación técnica como el propio Swagger: documenta el **por qué**, no solo el **qué**.

Dónde guardarlos: en el mismo repositorio, como `docs/adr/0007-postgresql-para-pagos.md`. Así se versionan con el código, aparecen en los PR y nadie tiene que buscar en otra herramienta.

Los ADRs no se editan cuando una decisión cambia: se crea uno nuevo que **reemplaza** al anterior (y el viejo queda marcado como "Reemplazado por ADR-0012"). Así queda el historial de por qué se cambió de opinión.

-----

## Ejemplo completo

El endpoint del problema, documentado de punta a punta.

`CrearPedidoDto.cs` (el contrato acordado con frontend antes de programar):

```csharp
/// <summary>Datos para crear un pedido.</summary>
/// <param name="CustomerId">Id del cliente. Debe existir.</param>
/// <param name="Items">Al menos un producto.</param>
/// <param name="CouponCode">Código de cupón opcional.</param>
public record CrearPedidoDto(
    [property: Range(1, int.MaxValue)] int CustomerId,
    [property: Required, MinLength(1)] List<ItemDto> Items,
    string? CouponCode);

public record ItemDto([property: Range(1, int.MaxValue)] int ProductId, [property: Range(1, 100)] int Quantity);

public record OrderDto(int Id, decimal Total, string Status);
```

`OrdersController.cs`:

```csharp
[ApiController]
[Route("api/v1/orders")]
public class OrdersController(OrderService service) : ControllerBase
{
    /// <summary>Crea un pedido para un cliente existente.</summary>
    /// <remarks>El total se calcula en el servidor a partir de los precios vigentes.</remarks>
    [HttpPost]
    public async Task<Results<Created<OrderDto>, ValidationProblem, UnprocessableEntity<ProblemDetails>>>
        Crear(CrearPedidoDto dto, CancellationToken ct)
    {
        var resultado = await service.CrearAsync(dto, ct);

        return resultado.Exito
            ? TypedResults.Created($"/api/v1/orders/{resultado.Pedido!.Id}", resultado.Pedido)
            : TypedResults.UnprocessableEntity(new ProblemDetails
            {
                Title = "No se pudo crear el pedido",
                Detail = resultado.Error,
                Status = StatusCodes.Status422UnprocessableEntity
            });
    }
}
```

`Program.cs` (.NET 9):

```csharp
builder.Services.AddControllers();
builder.Services.AddProblemDetails();
builder.Services.AddOpenApi();

var app = builder.Build();
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference();
}
app.MapControllers();
app.Run();
```

`docs/adr/0007-postgresql-para-pedidos.md`:

```markdown
# ADR-0007: PostgreSQL para pedidos, MongoDB para el catálogo

## Estado
Aceptada (2026-03-14)

## Contexto
Los pedidos y pagos necesitan transacciones ACID entre varias tablas.
El catálogo tiene atributos muy distintos por categoría de producto.

## Decisión
Pedidos y pagos en PostgreSQL. Catálogo de productos en MongoDB.

## Alternativas consideradas
- Todo en PostgreSQL con columnas JSONB para los atributos: más simple de operar,
  pero las búsquedas por atributo del catálogo eran 4 veces más lentas en la prueba.
- Todo en MongoDB: descartada por las transacciones de pagos.

## Consecuencias
- Dos motores que operar y respaldar.
- Un pedido guarda una copia del nombre y precio del producto (no hay JOIN entre motores).
```

Qué observar:

* La documentación muestra `201`, `400` y `422`, y no puede mentir: los resultados tipados obligan al método a devolver solo esos.
* `customerId` y `quantity` aparecen con su rango, porque los atributos de validación del DTO se reflejan en la especificación. Se usa `[Range]` y no `[Required]` porque un `int` nunca es `null`: si el campo falta en el JSON llega como `0`, y `[Required]` lo daría por válido.
* El DTO se acordó antes: el frontend trabajó con un mock desde el primer día.
* La pregunta "¿por qué dos bases de datos?" tiene una respuesta escrita, con las alternativas que se descartaron.
* La UI de la documentación solo se expone en Development.

-----

## Errores comunes

**1. Documentación que solo muestra `200`.**
Qué pasa: los consumidores descubren los otros códigos en producción.
Por qué: el método devuelve `IActionResult` sin atributos.
Arreglo: `ProducesResponseType` o, mejor, resultados tipados.

**2. Definir el contrato mientras se programa.**
Qué pasa: frontend y backend reescriben código por cambios en los DTOs.
Por qué: se empieza a programar antes de acordar qué se intercambia.
Arreglo: acuerda los DTOs (o un borrador de OpenAPI) al inicio y versiónalos.

**3. Renombrar campos "porque es el mismo dato".**
Qué pasa: un cliente en producción deja de funcionar.
Por qué: no se trata el renombre como un breaking change.
Arreglo: los cambios incompatibles van en una versión nueva de la API; mientras tanto, mantén el nombre viejo.

**4. Exponer la UI de documentación en producción sin protección.**
Qué pasa: cualquiera ve todos los endpoints internos, incluso los de administración.
Por qué: se deja `UseSwaggerUI()` o `MapScalarApiReference()` fuera del `if (IsDevelopment())`.
Arreglo: exponla solo en desarrollo o detrás de autenticación.

**5. ADRs en una wiki externa (o en ningún lado).**
Qué pasa: las decisiones se olvidan, o la wiki queda desactualizada respecto del código.
Por qué: la documentación vive lejos del código que describe.
Arreglo: ADRs en Markdown dentro del repositorio, revisados en los mismos PR.

-----

## Evolución y versiones

* **2011:** nace Swagger como especificación y conjunto de herramientas.
* **2015:** SmartBear dona la especificación a la Linux Foundation, que la renombra **OpenAPI**; "Swagger" queda como nombre de las herramientas.
* **2017:** OpenAPI 3.0.
* **2021:** OpenAPI 3.1, alineada con JSON Schema.
* **.NET 7:** resultados tipados (`Results<...>`, `TypedResults`).
* **.NET 9 (2024):** la plantilla deja de incluir Swashbuckle y usa `Microsoft.AspNetCore.OpenApi` (`AddOpenApi`, `MapOpenApi`).
* **.NET 10 (2025):** `Microsoft.AspNetCore.OpenApi` incorpora los comentarios XML a la especificación.

-----

## Cuándo sí y cuándo no

**Documenta con OpenAPI cuando:** la API la consume alguien que no la escribió: otro equipo, un frontend, terceros. Es decir, casi siempre.

**Usa contract-first cuando:** varios equipos trabajan en paralelo o la API es pública y el contrato es lo más importante.

**Escribe un ADR cuando:** la decisión es difícil de revertir o alguien nuevo podría preguntarse "¿por qué está hecho así?".

**No escribas un ADR cuando:** la decisión es trivial o fácil de revertir. Un ADR por cada librería de utilidades es ruido.

-----

## Resumen en 5 líneas

1. Tres documentaciones distintas: OpenAPI (cómo usar la API), contratos (qué se intercambia) y ADRs (por qué).
2. Desde .NET 9, la especificación se genera con `AddOpenApi()`; Swashbuckle ya no viene en la plantilla.
3. Documenta todos los códigos de respuesta, idealmente con resultados tipados para que la documentación no pueda mentir.
4. Acuerda los DTOs antes de programar; renombrar un campo es un breaking change.
5. Los ADRs van en el repositorio y no se editan: una decisión nueva reemplaza a la anterior.

-----

## Para profundizar

<details>
<summary>Generar clientes desde la especificación</summary>

Con la especificación OpenAPI, el frontend puede generar automáticamente un cliente tipado (por ejemplo, con `openapi-typescript` o NSwag), en lugar de escribir a mano cada llamada `fetch` y cada interfaz TypeScript:

```text
npx openapi-typescript https://localhost:5001/openapi/v1.json -o src/api/schema.d.ts
```

Si el backend cambia un campo, el frontend se entera **al compilar**, no en producción. Es la forma más concreta de que el contrato sea un contrato de verdad.

</details>

<details>
<summary>Validar que la API cumple el contrato en CI</summary>

En contract-first, la especificación es la fuente de verdad, y hay que comprobar que el código la cumple. Herramientas como Spectral validan que la especificación siga las reglas del equipo (nombres en plural, todas las respuestas documentadas), y otras comparan dos versiones de la especificación y fallan si hay un breaking change (por ejemplo, `oasdiff`). Integradas en el pipeline, convierten "acordamos no romper el contrato" en algo que el build hace cumplir.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Documento la API con Swagger, que genera una página donde se ven y se prueban los endpoints. Uso DTOs para definir qué datos recibe y devuelve cada endpoint.

### Respuesta ampliada (semi-senior)

Mantengo tres tipos de documentación. La especificación OpenAPI, que en .NET 9 genero con `AddOpenApi()`, documentando todos los códigos de respuesta, preferiblemente con resultados tipados para que no pueda desincronizarse del código. Los contratos: acuerdo los DTOs con el frontend antes de programar, para que puedan trabajar en paralelo con un mock, y trato cualquier renombre o eliminación de campos como un breaking change que va en una versión nueva. Y los ADRs, en el repositorio, para las decisiones difíciles de revertir, porque Swagger dice cómo se usa la API pero no por qué está hecha así.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre Swagger y OpenAPI?**
OpenAPI es la especificación. Swagger fue su nombre original y hoy es el nombre de un conjunto de herramientas, como Swagger UI.

**2. ¿Code-first o contract-first?**
Contract-first cuando varios equipos trabajan en paralelo o la API es pública. Code-first cuando el mismo equipo hace backend y frontend y la velocidad importa más. Un punto intermedio es acordar los DTOs al inicio aunque el código genere la especificación.

**3. ¿Qué cambios rompen el contrato de una API?**
Quitar o renombrar campos, cambiar sus tipos, hacer obligatorio algo que era opcional o cambiar el significado de un código de estado. Agregar campos opcionales o endpoints nuevos no lo rompe.

-----

## Práctica

**Ejercicio 1.** Este endpoint puede devolver `200`, `404` (el pedido no existe) y `403` (el pedido es de otro cliente). Documéntalo para que la especificación sea correcta.

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> Obtener(int id) { /* ... */ }
```

<details>
<summary>Solución</summary>

Con atributos:

```csharp
[HttpGet("{id:int}")]
[ProducesResponseType<OrderDto>(StatusCodes.Status200OK)]
[ProducesResponseType<ProblemDetails>(StatusCodes.Status403Forbidden)]
[ProducesResponseType<ProblemDetails>(StatusCodes.Status404NotFound)]
public async Task<IActionResult> Obtener(int id) { /* ... */ }
```

Mejor, con resultados tipados:

```csharp
[HttpGet("{id:int}")]
public async Task<Results<Ok<OrderDto>, ForbidHttpResult, NotFound>> Obtener(int id) { /* ... */ }
```

Con la segunda versión, si alguien agrega otro tipo de respuesta, el código no compila hasta que lo incluya en el tipo, y la documentación se actualiza sola.

</details>

**Ejercicio 2.** ¿Cuáles de estos cambios en `OrderDto` rompen el contrato?

1. Agregar el campo opcional `discountCode`.
2. Renombrar `total` a `totalAmount`.
3. Cambiar `status` de `string` a un número.
4. Agregar el endpoint `GET /api/v1/orders/{id}/invoice`.
5. Hacer obligatorio `couponCode` en `CrearPedidoDto`, que antes era opcional.

<details>
<summary>Solución</summary>

* **1. No rompe:** los clientes que no lo conocen lo ignoran.
* **2. Rompe:** los clientes leen `total`, que deja de existir.
* **3. Rompe:** los clientes esperan un texto.
* **4. No rompe:** es un endpoint nuevo.
* **5. Rompe:** las peticiones de los clientes existentes, que no lo envían, empiezan a fallar con `400`.

Los cambios 2, 3 y 5 necesitan una versión nueva (`/api/v2`) o un período de convivencia en el que se acepten las dos formas.

</details>

-----

## Siguiente lección

Terminaste las lecciones de esta fase. Para continuar: [Arquitectura frontend, rendimiento y calidad](../04-arquitectura-frontend-rendimiento-calidad/).
