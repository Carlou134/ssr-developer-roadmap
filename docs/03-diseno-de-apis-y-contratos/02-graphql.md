# GraphQL

## ¿Qué problema resuelve?

GraphQL es un lenguaje de consulta para APIs, creado por Meta (Facebook) en 2012. A diferencia de REST, donde hay múltiples endpoints (uno por recurso), GraphQL expone **un solo endpoint** (normalmente `/graphql`), y el cliente pide exactamente los campos que necesita.

**El problema que resuelve: over-fetching y under-fetching.**

- **Over-fetching:** la respuesta trae MÁS datos de los que hacen falta. Ejemplo: se pide el perfil de un usuario y la API devuelve 20 campos, pero la pantalla solo muestra 3 (nombre, foto, bio) — el resto viajó por la red sin usarse para nada.
- **Under-fetching:** una sola respuesta NO alcanza, y hace falta encadenar varias peticiones para juntar todo lo que la pantalla necesita. Ejemplo: para mostrar el perfil de un usuario CON sus publicaciones, en REST podrían hacer falta 2-3 llamadas distintas (una para el usuario, otra para sus posts).

GraphQL ataca los dos problemas a la vez: el cliente arma una sola consulta pidiendo exactamente los campos que necesita, de todos los recursos relacionados que hagan falta, y el servidor responde con eso — ni más, ni menos, en una sola ida y vuelta.

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

## ¿Cuándo conviene GraphQL sobre REST?

Es un trade-off, no una mejora universal — GraphQL no reemplaza a REST en todos los casos, y trae sus propios costos reales.

### La tabla de decisión

| Usar REST | Usar GraphQL |
|---|---|
| Los recursos están bien definidos y estables | Hay varios clientes (web, iOS, Android) pidiendo subconjuntos distintos de los mismos datos |
| Los clientes necesitan más o menos los mismos datos | Las pantallas piden combinaciones muy variables de datos relacionados entre sí |
| El caching HTTP estándar (CDN, navegador, proxies) importa | El over-fetching es un problema medido, no una sospecha |
| Es una API interna de negocio | Es una plataforma con API pública para terceros, con necesidades impredecibles |

En la práctica, la mayoría de las APIs de negocio (como una API típica de gestión, del estilo OrderManagementBackend) caen del lado izquierdo. Por eso REST sigue siendo, por lejos, la opción más común.

### Ejemplos reales de quién usa GraphQL

- **GitHub** — su API v4 es GraphQL completa (la v3, en REST, sigue viva en paralelo).
- **Shopify** — la Admin API y la Storefront API son GraphQL.
- **Meta/Facebook** — lo crearon, y lo usan internamente para sus propias apps.

El patrón detrás de los tres casos es siempre el mismo: **muchos clientes distintos pidiendo subconjuntos distintos de los mismos datos.**

### Los costos reales de elegir GraphQL

- **El problema N+1** (ver el checklist de punto 6): si el resolver de `posts` hace una consulta a la base de datos por cada usuario de una lista, 100 usuarios significan 100 consultas separadas. La solución estándar es el patrón **DataLoader**, que agrupa esas consultas en lotes — pero GraphQL no lo previene solo, hay que diseñarlo a propósito.
- **El caching HTTP estándar se pierde.** REST se apoya en el verbo `GET` y la URL para cachear (a nivel de CDN, navegador, proxies). GraphQL, al usar típicamente un solo endpoint vía `POST`, pierde ese cacheo automático — hay que resolverlo con herramientas específicas de GraphQL.
