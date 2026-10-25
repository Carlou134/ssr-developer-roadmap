# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**ADR (*Architecture Decision Record*).** Documento corto que registra una decisión técnica, sus alternativas y por qué se eligió. Se guarda en el repositorio.

**Breaking change.** Cambio en el contrato que rompe a los clientes existentes, como quitar o renombrar un campo.

**Contract-first / code-first.** Escribir primero el contrato (la especificación OpenAPI) y generar el código, o escribir el código y generar el contrato.

**Contrato.** El acuerdo sobre qué recibe y qué devuelve una API, en qué formato y con qué errores.

**Cursor (paginación por).** Paginar indicando el último elemento visto en lugar de un número de página. Es estable y rápido en listas grandes.

**DataLoader.** Patrón que agrupa en una sola consulta las cargas que se pedirían una por una. Es la solución estándar al N+1 en GraphQL.

**ETag.** Identificador de la versión de un recurso, usado para el caché y para evitar que dos clientes se pisen los cambios.

**GraphQL.** Lenguaje de consulta para APIs en el que el cliente describe la forma exacta de los datos que quiere, contra un único endpoint.

**HATEOAS (*Hypermedia as the Engine of Application State*).** Que la respuesta incluya links a las acciones disponibles desde ahí. Es el nivel 3 de Richardson y casi nadie lo implementa.

**Idempotency key (clave de idempotencia).** Identificador único que el cliente envía en un `POST` para que el servidor no repita la operación si recibe un reintento.

**Idempotente.** Que ejecutar la operación una vez o muchas veces deja el mismo estado final. `GET`, `PUT` y `DELETE` lo son; `POST`, no.

**Método seguro.** Método HTTP que no modifica el estado del servidor, como `GET`.

**MIME type (*media type*).** Identificador del formato de un contenido, como `application/json` o `application/problem+json`.

**Modelo de madurez de Richardson.** Cuatro niveles de "qué tan REST" es una API: 0 (un endpoint RPC), 1 (recursos), 2 (verbos y códigos), 3 (HATEOAS).

**Mutation.** Operación de GraphQL que modifica datos.

**Offset (paginación por).** Paginar con número de página y tamaño (`?page=3&pageSize=20`). Simple, pero lento en páginas lejanas.

**OpenAPI.** Especificación estándar para describir APIs HTTP en un archivo JSON o YAML.

**Over-fetching.** La respuesta trae más datos de los que se necesitan.

**Paginación.** Devolver una lista en partes (páginas) en lugar de completa.

**Persisted query.** Consulta de GraphQL registrada de antemano en el servidor, que el cliente invoca por un identificador.

**Problem Details.** Formato estándar para describir errores en APIs HTTP, definido en RFC 9457 (antes RFC 7807).

**Problema N+1.** Una consulta para obtener N elementos, más una consulta adicional por cada uno de ellos.

**Query (GraphQL).** Operación de GraphQL que lee datos.

**Recurso.** Cualquier cosa que la API expone y que tiene una identidad, como un pedido o un cliente.

**Representación.** El formato en que viaja un recurso entre cliente y servidor, normalmente JSON.

**Resolver.** La función del servidor GraphQL que obtiene el valor de un campo.

**REST (*Representational State Transfer*).** Estilo arquitectónico para sistemas en red, definido por Roy Fielding en el año 2000 como un conjunto de restricciones.

**Resultados tipados.** En ASP.NET Core, un tipo de retorno como `Results<Ok<T>, NotFound>` que declara qué respuestas puede devolver un endpoint, y que el compilador verifica.

**Schema (GraphQL).** La definición tipada de todo lo que la API expone: tipos, campos y operaciones.

**Stateless (sin estado).** El servidor no recuerda peticiones anteriores: cada petición trae todo lo que necesita.

**Subscription.** Operación de GraphQL que recibe datos en tiempo real cuando algo cambia.

**Swagger.** El nombre original de OpenAPI (hasta 2015) y hoy el de un conjunto de herramientas, como Swagger UI.

**Swashbuckle.** Librería de .NET que genera la especificación OpenAPI y sirve Swagger UI. Desde .NET 9 ya no viene en la plantilla.

**Under-fetching.** Una respuesta no alcanza y hay que encadenar varias peticiones.

**URI.** El identificador de un recurso, como `/api/v1/orders/12`.

**Versionado.** Publicar cambios incompatibles en una versión nueva de la API (`/api/v2`) para no romper a los clientes existentes.
