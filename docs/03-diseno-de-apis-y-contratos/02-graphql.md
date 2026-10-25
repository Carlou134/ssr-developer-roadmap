# GraphQL

## En una frase

GraphQL es un lenguaje de consulta para APIs con **un solo endpoint donde el cliente pide exactamente los campos que necesita**, de varios recursos relacionados, en una sola ida y vuelta; resuelve el over-fetching y el under-fetching, a cambio de perder el caché HTTP estándar y de tener que cuidar a mano el N+1 y el costo de las consultas.

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo funciona una API REST: [Estándares RESTful](01-estandares-restful.md).
* Qué es un trade-off: [¿Qué son los trade-offs?](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **GraphQL:** lenguaje de consulta para APIs en el que el cliente describe la forma exacta de los datos que quiere.
* **Over-fetching:** la respuesta trae más datos de los que se necesitan.
* **Under-fetching:** una respuesta no alcanza y hay que encadenar varias peticiones.
* **Schema:** la definición tipada de todo lo que la API expone: tipos, campos y operaciones.
* **Query:** operación de GraphQL que lee datos.
* **Mutation:** operación de GraphQL que modifica datos.
* **Subscription:** operación de GraphQL que recibe datos en tiempo real cuando algo cambia.
* **Resolver:** la función del servidor que obtiene el valor de un campo.
* **Problema N+1:** una consulta para obtener N elementos, más una consulta adicional por cada uno de ellos.
* **DataLoader:** patrón que agrupa en una sola consulta las cargas que se pedirían una por una.

-----

## El problema

La pantalla de perfil de una app móvil muestra el nombre y la foto de un usuario, y los títulos de sus últimos 5 posts. Con la API REST existente:

```http
GET /api/v1/users/123
→ 32 campos (dirección, preferencias, fechas de auditoría...), la pantalla usa 2

GET /api/v1/users/123/posts?pageSize=5
→ cada post con su contenido completo, la pantalla usa solo el título

GET /api/v1/users/123/followers/count
→ un número
```

Tres viajes de red en una conexión móvil lenta, y la mayoría de los bytes descargados no se usan. La versión web necesita otros campos, así que el equipo de backend agrega `GET /users/123/profile-mobile` y `GET /users/123/profile-web`.

Las consecuencias:

* **under-fetching:** tres peticiones encadenadas para una sola pantalla;
* **over-fetching:** datos que viajan por la red sin usarse;
* el backend termina creando endpoints a medida para cada pantalla de cada cliente.

-----

## Cómo funciona

### 1. Qué es y qué problema resuelve

GraphQL es un lenguaje de consulta para APIs que Meta (entonces Facebook) creó en 2012 para sus apps móviles y publicó como código abierto en 2015. A diferencia de REST, donde hay un endpoint por recurso, GraphQL expone **un solo endpoint** (normalmente `/graphql`), y el cliente describe exactamente qué campos quiere.

```graphql
query {
  user(id: "123") {
    name
    avatarUrl
    followersCount
    posts(last: 5) {
      title
    }
  }
}
```

```json
{
  "data": {
    "user": {
      "name": "Ana",
      "avatarUrl": "https://cdn.app.com/ana.jpg",
      "followersCount": 842,
      "posts": [
        { "title": "Primer post" },
        { "title": "Segundo post" }
      ]
    }
  }
}
```

La respuesta tiene **la misma forma que la consulta**, ni un campo más. Ataca los dos problemas a la vez:

| Problema | En REST | En GraphQL |
| --- | --- | --- |
| **Over-fetching** | El endpoint devuelve todos sus campos | El cliente pide solo los campos que usa |
| **Under-fetching** | Varias peticiones para juntar recursos relacionados | Una consulta con los recursos anidados |

### 2. El schema: el contrato tipado

Todo lo que la API expone se define en un **schema** con tipos. Es el contrato entre cliente y servidor, y se puede consultar (*introspección*), lo que permite que las herramientas autocompleten las consultas.

```graphql
type User {
  id: ID!
  name: String!
  avatarUrl: String
  followersCount: Int!
  posts(last: Int): [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
}

type Query {
  user(id: ID!): User
}
```

El `!` significa "no puede ser `null`". `[Post!]!` es una lista que nunca es `null` y cuyos elementos tampoco lo son.

### 3. Los tres tipos de operación

| Operación | Equivale en REST a... | Para qué |
| --- | --- | --- |
| **Query** | `GET` | Leer datos sin modificar nada |
| **Mutation** | `POST`, `PUT`, `PATCH`, `DELETE` | Modificar datos |
| **Subscription** | (no hay equivalente directo; WebSockets o SSE) | Recibir datos en tiempo real cuando algo cambia |

```graphql
mutation {
  createPost(input: { title: "Nuevo post", content: "..." }) {
    id
    title
  }
}
```

La mutation también elige qué campos devolver del resultado.

### 4. Resolvers: un resolver por campo

En el servidor, cada campo del schema tiene un **resolver**: la función que obtiene su valor. No hay un resolver por endpoint, sino uno por campo. Para la mayoría de los campos el resolver es trivial (leer una propiedad del objeto); los interesantes son los que cargan datos relacionados, como `posts` dentro de `User`.

En .NET, la librería más usada es **Hot Chocolate**:

```csharp
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddTypeExtension<UserExtensions>();

app.MapGraphQL(); // expone /graphql

public class Query
{
    public Task<User?> GetUser(int id, AppDbContext db) =>
        db.Users.FirstOrDefaultAsync(u => u.Id == id);
}

[ExtendObjectType<User>]
public class UserExtensions
{
    // resolver del campo "posts" de User
    public Task<List<Post>> GetPosts([Parent] User user, int last, AppDbContext db) =>
        db.Posts.Where(p => p.UserId == user.Id)
                .OrderByDescending(p => p.CreatedAt)
                .Take(last)
                .ToListAsync();
}
```

Hot Chocolate convierte `GetUser` en el campo `user` y genera el schema a partir de las clases de C#.

### 5. El problema N+1

Es el costo más conocido de GraphQL. Con el resolver anterior, esta consulta:

```graphql
query {
  users(first: 100) {
    name
    posts(last: 3) { title }
  }
}
```

ejecuta **1** consulta para los 100 usuarios y después **100** consultas más, una por cada usuario, para sus posts: 101 viajes a la base de datos. El problema N+1 se ve con más detalle en la fase de bases de datos.

La solución estándar es el patrón **DataLoader**: en lugar de cargar los posts de cada usuario por separado, el DataLoader acumula todos los Ids que se piden durante la ejecución y hace **una sola consulta** para todos.

```csharp
// Hot Chocolate 14: el generador de código crea IPostsByUserIdDataLoader
public static class PostDataLoaders
{
    [DataLoader]
    public static async Task<ILookup<int, Post>> GetPostsByUserIdAsync(
        IReadOnlyList<int> userIds, AppDbContext db, CancellationToken ct)
    {
        var posts = await db.Posts.Where(p => userIds.Contains(p.UserId)).ToListAsync(ct);
        return posts.ToLookup(p => p.UserId);
    }
}

[ExtendObjectType<User>]
public class UserExtensions
{
    public Task<Post[]> GetPosts([Parent] User user, IPostsByUserIdDataLoader loader, CancellationToken ct) =>
        loader.LoadAsync(user.Id, ct);
}
```

Ahora la misma consulta ejecuta **2** consultas a la base de datos, sin importar cuántos usuarios haya. GraphQL no previene el N+1 solo: hay que diseñarlo a propósito.

### 6. Los errores no siempre son un código HTTP

En REST, el código de estado dice qué pasó. En GraphQL, una consulta con errores parciales normalmente responde **`200 OK`** con los datos que sí se pudieron obtener y una lista de `errors`:

```json
{
  "data": {
    "user": { "name": "Ana", "posts": null }
  },
  "errors": [
    {
      "message": "No se pudieron cargar los posts",
      "path": ["user", "posts"],
      "extensions": { "code": "SERVICE_UNAVAILABLE" }
    }
  ]
}
```

Si lo usas mal:

```javascript
const respuesta = await fetch("/graphql", { method: "POST", body });
if (respuesta.ok) mostrar(respuesta.json().data); // ¿y los errores?
```

Un cliente que solo revisa `respuesta.ok` (el código HTTP) no se entera de que una parte falló. Con GraphQL hay que revisar **siempre** el campo `errors`. Y las herramientas de monitoreo que cuentan errores por código HTTP mostrarán 0 % de fallos.

### 7. Cuándo conviene sobre REST

Es un trade-off, no una mejora universal.

| Usa REST cuando... | Usa GraphQL cuando... |
| --- | --- |
| Los recursos están bien definidos y son estables | Hay varios clientes (web, iOS, Android) que piden subconjuntos distintos de los mismos datos |
| Los clientes necesitan más o menos los mismos datos | Las pantallas combinan datos relacionados de formas muy variables |
| El caché HTTP estándar (CDN, navegador, proxies) importa | El over-fetching es un problema **medido**, no una sospecha |
| Es una API interna de negocio | Es una plataforma con API pública para terceros, con necesidades impredecibles |

La mayoría de las APIs de negocio (como una API de gestión del estilo de mi `OrderManagementBackend`) caen del lado izquierdo. Por eso REST sigue siendo, por lejos, la opción más común.

**Quién usa GraphQL:**

* **GitHub:** su API v4 es GraphQL (la v3, REST, sigue en paralelo).
* **Shopify:** sus APIs Admin y Storefront son GraphQL.
* **Meta:** lo creó para sus propias apps.

El patrón es siempre el mismo: **muchos clientes distintos pidiendo subconjuntos distintos de los mismos datos.**

### 8. Los costos reales de elegir GraphQL

| Costo | Por qué | Cómo se mitiga |
| --- | --- | --- |
| **N+1** | Los resolvers cargan datos relacionados uno por uno | DataLoader |
| **Se pierde el caché HTTP** | Todo va por `POST /graphql`: la URL no identifica la consulta, así que CDN y navegador no pueden cachear | Caché en el cliente (Apollo, Relay), *persisted queries* por `GET` |
| **Consultas costosas o maliciosas** | El cliente arma la consulta: puede anidar `users → posts → author → posts → ...` hasta tirar el servidor | Límites de profundidad, de complejidad y de tiempo de ejecución |
| **Errores con `200 OK`** | Los errores parciales van en el body | Revisar siempre `errors`; monitoreo específico |
| **Autorización por campo** | Cualquier campo puede pedirse desde cualquier consulta | Reglas de autorización en el schema o en cada resolver |

-----

## Ejemplo completo

La pantalla de perfil del problema, con GraphQL.

`Program.cs`:

```csharp
builder.Services.AddDbContext<AppDbContext>(o => o.UseSqlServer(connectionString));

builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddTypeExtension<UserExtensions>()   // el DataLoader generado se registra solo
    .ModifyRequestOptions(o => o.ExecutionTimeout = TimeSpan.FromSeconds(10))
    .AddMaxExecutionDepthRule(5); // nadie puede anidar más de 5 niveles

var app = builder.Build();
app.MapGraphQL();
app.Run();
```

`Query.cs` y `UserExtensions.cs`:

```csharp
public class Query
{
    public Task<User?> GetUser(int id, AppDbContext db, CancellationToken ct) =>
        db.Users.FirstOrDefaultAsync(u => u.Id == id, ct);
}

[ExtendObjectType<User>]
public class UserExtensions
{
    public async Task<Post[]> GetPosts([Parent] User user, int last,
        IPostsByUserIdDataLoader loader, CancellationToken ct) =>
        (await loader.LoadAsync(user.Id, ct)).OrderByDescending(p => p.CreatedAt).Take(last).ToArray();

    public Task<int> GetFollowersCount([Parent] User user, AppDbContext db, CancellationToken ct) =>
        db.Follows.CountAsync(f => f.FollowedId == user.Id, ct);
}
```

Consulta de la app móvil:

```graphql
query PerfilMovil($id: Int!) {
  user(id: $id) {
    name
    avatarUrl
    followersCount
    posts(last: 5) { title }
  }
}
```

Consulta de la web, al **mismo** endpoint, con otros campos:

```graphql
query PerfilWeb($id: Int!) {
  user(id: $id) {
    name
    avatarUrl
    bio
    posts(last: 10) { title content createdAt }
  }
}
```

Qué observar:

* Un viaje de red por pantalla en lugar de tres.
* Ya no hacen falta `/profile-mobile` ni `/profile-web`: cada cliente pide lo suyo.
* Las variables (`$id`) separan la consulta de los datos: la consulta es fija y se puede registrar como *persisted query*.
* La regla de profundidad máxima y el timeout protegen al servidor de consultas abusivas.
* Con el DataLoader, pedir 100 usuarios con sus posts son dos consultas, no 101.

-----

## Errores comunes

**1. Elegir GraphQL "porque es más moderno".**
Qué pasa: complejidad extra (schema, DataLoaders, límites, caché del cliente) para una API que tiene un solo cliente.
Por qué: se elige la tecnología antes de medir el problema.
Arreglo: usa GraphQL cuando haya varios clientes con necesidades distintas o un over-fetching medido.

**2. Resolvers sin DataLoader.**
Qué pasa: una consulta que en desarrollo tarda 50 ms y en producción, con 500 elementos, tarda 8 segundos.
Por qué: cada elemento de la lista dispara su propia consulta (N+1).
Arreglo: DataLoader para todo campo que cargue datos relacionados.

**3. No limitar la profundidad ni la complejidad.**
Qué pasa: una consulta anidada de forma recursiva consume toda la CPU o la memoria del servidor.
Por qué: el servidor ejecuta lo que el cliente pida.
Arreglo: límites de profundidad, de complejidad y de tiempo de ejecución.

**4. Ignorar el campo `errors`.**
Qué pasa: la pantalla muestra datos incompletos sin avisar, y el monitoreo no ve los fallos.
Por qué: se revisa solo el código HTTP, que suele ser `200`.
Arreglo: el cliente revisa `errors` en cada respuesta, y el monitoreo cuenta los errores de GraphQL.

**5. Exponer el modelo de la base de datos tal cual.**
Qué pasa: el schema cambia cada vez que cambia una tabla, y aparecen campos internos expuestos.
Por qué: se generan los tipos directamente desde las entidades de EF Core.
Arreglo: diseña el schema desde lo que necesitan los clientes, igual que un DTO.

-----

## Evolución y versiones

* **2012:** Facebook crea GraphQL para sus apps móviles.
* **2015:** se publica la especificación y la implementación de referencia como código abierto.
* **2016:** GitHub lanza su API v4 en GraphQL.
* **2018:** GraphQL pasa a la GraphQL Foundation, bajo la Linux Foundation.
* **.NET:** Hot Chocolate (ChilliCream) se consolida como la implementación más usada; la versión 14 (2024) agrega DataLoaders generados por código fuente.

-----

## Cuándo sí y cuándo no

**Usa GraphQL cuando:** varios clientes consumen los mismos datos con necesidades distintas, las pantallas combinan muchos recursos relacionados o ofreces una API pública con usos impredecibles.

**No uses GraphQL cuando:** la API tiene un solo cliente con necesidades estables, el caché HTTP en CDN es clave para el rendimiento, es una API simple de negocio, o el equipo no tiene tiempo para cuidar el N+1, los límites y el monitoreo. Ahí REST es más simple y más barato.

-----

## Resumen en 5 líneas

1. GraphQL expone un solo endpoint y el cliente pide exactamente los campos que necesita.
2. Resuelve el over-fetching y el under-fetching en una sola ida y vuelta.
3. Tiene tres operaciones (query, mutation, subscription) y un resolver por campo, sobre un schema tipado.
4. Sus costos: N+1 (se resuelve con DataLoader), pérdida del caché HTTP, consultas abusivas y errores con `200 OK`.
5. Conviene con muchos clientes pidiendo subconjuntos distintos de los mismos datos; para el resto, REST.

-----

## Para profundizar

<details>
<summary>Persisted queries: recuperar el caché y la seguridad</summary>

En lugar de enviar el texto completo de la consulta, el cliente envía un identificador (normalmente el hash de la consulta), que el servidor ya conoce:

```http
GET /graphql?id=a3f5c9...&variables={"id":123}
```

Como es un `GET` con una URL estable, vuelve a funcionar el caché HTTP en CDN y navegador. Y si el servidor **solo** acepta consultas registradas de antemano, nadie puede enviar consultas arbitrarias: desaparece el riesgo de las consultas abusivas en APIs que no son públicas.

</details>

<details>
<summary>Federation: un schema, varios servicios</summary>

En arquitecturas de microservicios, cada servicio puede exponer su parte del schema (usuarios, pedidos, productos), y un *gateway* las une en un único grafo. El cliente ve una sola API, aunque detrás haya varios equipos y servicios. Apollo Federation y Hot Chocolate Fusion son las implementaciones más conocidas. Es potente, pero agrega bastante complejidad operativa: solo se justifica con varios equipos y servicios.

</details>

-----

## En entrevista

### Respuesta corta (junior)

GraphQL es un lenguaje de consulta para APIs donde el cliente pide exactamente los datos que necesita a un único endpoint. Tiene queries para leer y mutations para modificar, y evita traer datos de más o tener que hacer varias peticiones.

### Respuesta ampliada (semi-senior)

GraphQL resuelve el over-fetching y el under-fetching: el cliente describe la forma de los datos sobre un schema tipado y recibe exactamente eso en una sola ida y vuelta. Lo considero cuando hay varios clientes que piden subconjuntos distintos de los mismos datos, como web y móvil. Sé que tiene costos: hay que prevenir el N+1 con DataLoaders, se pierde el caché HTTP estándar, hay que limitar la profundidad y la complejidad de las consultas, y los errores parciales vienen con `200 OK`, así que el cliente y el monitoreo tienen que revisar el campo `errors`. Para una API de negocio con un solo cliente, prefiero REST.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es el problema N+1 en GraphQL y cómo se resuelve?**
Cuando un resolver carga datos relacionados por cada elemento de una lista: una consulta para la lista y N más. Se resuelve con DataLoader, que agrupa todas esas cargas en una sola consulta.

**2. ¿Por qué GraphQL no aprovecha el caché HTTP?**
Porque normalmente todas las consultas van por `POST` al mismo endpoint, y la URL no identifica qué se pide. Se mitiga con caché en el cliente o con persisted queries por `GET`.

**3. ¿GraphQL reemplaza a REST?**
No. Es un trade-off: gana flexibilidad para los clientes y pierde simplicidad, caché y control del costo de las consultas.

-----

## Práctica

**Ejercicio 1.** Escribe una consulta GraphQL que obtenga el nombre de un cliente con Id `42`, y de sus últimos 3 pedidos solo el total y la fecha.

<details>
<summary>Solución</summary>

```graphql
query PedidosDeCliente($id: ID!) {
  customer(id: $id) {
    name
    orders(last: 3) {
      total
      createdAt
    }
  }
}
```

Con las variables `{ "id": "42" }`. Usar variables en lugar de escribir el `42` dentro de la consulta permite reutilizarla y registrarla como persisted query.

</details>

**Ejercicio 2.** Un equipo propone migrar a GraphQL una API interna de facturación que consume un único frontend web, con pantallas que casi no cambian. ¿Qué le responderías?

<details>
<summary>Solución</summary>

Que el problema que resuelve GraphQL (varios clientes con necesidades distintas, over-fetching medido) no está presente: hay un solo cliente y sus necesidades son estables. A cambio, pagarían todos los costos: schema, DataLoaders para evitar el N+1, límites de complejidad, pérdida del caché HTTP y un monitoreo distinto de errores. Si hay un endpoint concreto con over-fetching, se puede resolver en REST con un DTO más chico o con un parámetro `fields`. La pregunta para el equipo: **¿qué problema medido tenemos hoy que GraphQL resuelve?**

</details>

-----

## Siguiente lección

[Documentación técnica](03-documentacion-tecnica.md)
