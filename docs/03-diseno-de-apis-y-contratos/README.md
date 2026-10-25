# Fase 3: Diseño de APIs y contratos técnicos

En esta carpeta aprendes a **diseñar una API como un contrato predecible**: recursos, verbos y códigos de estado con sentido, errores en formato estándar, paginación y versionado; cuándo GraphQL resuelve un problema real y cuánto cuesta; y cómo documentar el cómo (OpenAPI), el qué (contratos) y el por qué (ADRs).

-----

## Por qué importa

Una API es un contrato con quien la consume: un frontend, una app móvil u otro servicio. Ese contrato tiene que ser predecible: los mismos verbos para las mismas operaciones, los mismos códigos de estado para los mismos resultados y la misma estructura de error siempre. Sin esa consistencia, cada consumidor tiene que adivinar cómo se comporta la API, leyendo el código en lugar de la documentación.

-----

## Antes de empezar

Conviene que ya tengas:

* Lo básico de HTTP: petición, respuesta, headers, body y códigos de estado.
* Haber creado un controller en ASP.NET Core.
* Qué es un DTO y por qué no exponer entidades: [Arquitectura en capas](../02-calidad-de-codigo-y-arquitectura/08-arquitectura-en-capas-ntier.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Estándares RESTful](01-estandares-restful.md) | Las restricciones de REST y el modelo de Richardson; verbos, idempotencia, URIs, códigos de estado, Problem Details, paginación y versionado | HTTP básico |
| [2. GraphQL](02-graphql.md) | Over-fetching y under-fetching, schema, queries, mutations y subscriptions, N+1 con DataLoader y cuándo conviene sobre REST | Estándares RESTful |
| [3. Documentación técnica](03-documentacion-tecnica.md) | OpenAPI en .NET 9 y 10, resultados tipados, contratos antes de programar, breaking changes y ADRs | Estándares RESTful, DTOs |

-----

## El mapa completo en una mirada

```text
REST (lo más común)
  Recurso      → sustantivo en plural:      /api/v1/orders/12
  Acción       → verbo HTTP:                GET lee · POST crea · PUT reemplaza · PATCH modifica · DELETE borra
  Reintento    → seguro solo si es idempotente (GET, PUT, DELETE)
  Resultado    → código que dice la verdad: 201 · 204 · 400/422 · 401/403 · 404 · 409 · 429
  Error        → Problem Details (RFC 9457), siempre el mismo formato
  Listas       → siempre paginadas, con un máximo en el servidor
  Cambios      → los que rompen el contrato van en una versión nueva

GraphQL (cuando hay muchos clientes con necesidades distintas)
  Un endpoint, el cliente elige los campos · costo: N+1, caché HTTP, límites, errores con 200

Documentación
  OpenAPI      → CÓMO se usa       (AddOpenApi, resultados tipados)
  Contratos    → QUÉ se intercambia (DTOs acordados antes de programar)
  ADRs         → POR QUÉ está así   (en el repositorio)
```

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura, así sabes dónde buscar cada cosa:

1. **En una frase:** qué es, sin jerga.
2. **Antes de empezar:** qué tienes que saber y las palabras nuevas.
3. **El problema:** una situación concreta que muestra por qué existe la herramienta.
4. **Cómo funciona:** el paso a paso, con ejemplos cortos y explicados.
5. **Ejemplo completo:** todo junto, aplicado al mismo problema.
6. **Errores comunes:** lo que suele salir mal, por qué pasa y cómo se arregla.
7. **Evolución y versiones:** qué cambió con el tiempo.
8. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
9. **Resumen en 5 líneas:** para repasar.
10. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento.
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

-----

## Después de esta carpeta

Ya sabes diseñar el contrato del lado del servidor. El siguiente paso es el otro lado: cómo lo consume y lo gestiona el cliente en [Arquitectura frontend, rendimiento y calidad](../04-arquitectura-frontend-rendimiento-calidad/).
