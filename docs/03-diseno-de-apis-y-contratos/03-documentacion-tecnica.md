# Documentación Técnica

Swagger/OpenAPI resuelve la documentación para PROBAR la API, pero un Semi-Senior mantiene tres tipos de documentación distintos, no solo ese uno.

## 1. Documentación de API (OpenAPI / Swagger)

En .NET, esto normalmente lo genera **Swashbuckle**, leyendo atributos y comentarios XML de los controladores. Por defecto, Swagger muestra un `200 OK` genérico — para que sea realmente útil, hay que decorar explícitamente los demás códigos posibles:

```csharp
[HttpPost]
[ProducesResponseType(typeof(OrderDto), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status422UnprocessableEntity)]
public IActionResult CrearPedido([FromBody] CrearPedidoDto dto)
{
    // ...
}
```

Sin esto, quien consuma la API (otro equipo, el frontend) solo ve "200 OK" en la documentación, y descubre los demás casos recién en producción.

## 2. Contratos e Interfaces (DTOs)

Definir los DTOs (Data Transfer Objects) entre Frontend y Backend **antes** de empezar a programar, no mientras se programa. Un DTO mal definido a mitad de camino obliga a ambos equipos a reescribir código en paralelo.

Esto también evita exponer las entidades de dominio directamente en la API — el DTO es la superficie pública del contrato, la entidad de dominio es un detalle interno que puede cambiar sin romper a nadie afuera.

## 3. Decisiones de Arquitectura (ADRs)

Ya se documentó a fondo en punto 1: ver [¿Qué son los Trade-offs?](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md), sección de Architecture Decision Records. Un ADR corto explicando por qué se tomó una decisión (ej: "se eligió PostgreSQL sobre MongoDB por la necesidad de transacciones ACID estrictas en el módulo de pagos") es tan parte de la documentación técnica como el propio Swagger — documenta el POR QUÉ, no solo el QUÉ.
