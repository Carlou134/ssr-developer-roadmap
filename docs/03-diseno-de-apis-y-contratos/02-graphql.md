# GraphQL

## ¿Qué problema resuelve?

GraphQL es un lenguaje de consulta para APIs, creado por Meta (Facebook) en 2012. A diferencia de REST, donde hay múltiples endpoints (uno por recurso), GraphQL expone **un solo endpoint** (normalmente `/graphql`), y el cliente pide exactamente los campos que necesita.

**El problema que resuelve: over-fetching y under-fetching.** En REST, para mostrar el perfil de un usuario con sus publicaciones, podrían hacer falta 2-3 peticiones distintas, o una sola que devuelva un JSON gigante con más campos de los que en realidad se necesitan.

```graphql
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}
```

El servidor responde únicamente con esos campos — ni uno más.

## Conceptos clave

- **Query:** equivalente a `GET` en REST — pedir datos sin modificar nada.
- **Mutation:** equivalente a `POST`/`PUT`/`DELETE` — modificar datos.
- **Resolver:** la función en el backend que resuelve cada campo solicitado — hay un resolver por campo, no por endpoint.

## ¿Cuándo conviene sobre REST? (es un trade-off, no una mejora universal)

GraphQL no reemplaza a REST en todos los casos — trae sus propios costos:

- **El problema N+1** (ver el checklist de punto 5) **es muy común en resolvers mal diseñados.** Si el resolver de `posts` hace una consulta a la base de datos por cada usuario de una lista, 100 usuarios significan 100 consultas separadas. La solución estándar es el patrón **DataLoader**, que agrupa esas consultas en lotes — pero hay que saber que el problema existe y diseñarlo a propósito, GraphQL no lo previene solo.
- **El caching HTTP estándar no funciona igual.** REST se apoya en el verbo `GET` y la URL para cachear (a nivel de CDN, navegador, proxies). GraphQL, al usar típicamente un solo endpoint vía `POST`, pierde ese cacheo automático — hay que resolverlo con herramientas específicas de GraphQL.
- **Conviene cuando:** el cliente necesita combinaciones de datos muy variables (distintas pantallas piden distintos subconjuntos de los mismos datos), o cuando hay varios clientes distintos (web, móvil) con necesidades de datos diferentes sobre las mismas fuentes.
- **REST sigue siendo la opción más simple** cuando los recursos están bien definidos y las necesidades de datos no varían mucho entre consumidores — que es el caso de la mayoría de las APIs internas de negocio.
