# Rate limiting

## En una frase

El rate limiting **limita cuántas peticiones puede hacer cada cliente en un período de tiempo**, para proteger la disponibilidad frente a abusos, frenar la fuerza bruta y repartir los recursos de forma justa; en ASP.NET Core viene integrado desde .NET 7, y la clave está en **por qué identificas al cliente** y en responder `429` con `Retry-After`.

-----

## Antes de empezar

Conviene que ya sepas:

* El código `429 Too Many Requests` y el header `Retry-After`: [Estándares RESTful](../03-diseno-de-apis-y-contratos/01-estandares-restful.md).
* Los ataques de fuerza bruta: [Autenticación y datos sensibles](05-autenticacion-y-datos-sensibles.md).
* Resiliencia del lado del cliente: [Resiliencia con Polly](07-resiliencia-con-polly.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Rate limiting (limitación de tasa):** limitar cuántas peticiones acepta el servidor por cliente en un período.
* **Throttling:** reducir la velocidad de procesamiento en lugar de rechazar; a menudo se usa como sinónimo de rate limiting.
* **Partición:** el criterio con el que se agrupan las peticiones para contarlas: por usuario, por IP, por API key.
* **Ventana fija (*fixed window*):** cuenta las peticiones en intervalos fijos (por ejemplo, cada minuto exacto).
* **Ventana deslizante (*sliding window*):** cuenta las peticiones de los últimos N segundos, dividiendo la ventana en segmentos.
* **Token bucket:** un balde que se recarga a ritmo constante; cada petición consume un token; permite ráfagas cortas.
* **Límite de concurrencia:** cuántas peticiones pueden estar **en proceso** al mismo tiempo.
* **Cuota:** límite de uso en un período largo (por ejemplo, 10 000 llamadas por mes en un plan gratuito).
* **Proxy inverso:** servidor que recibe las peticiones y las reenvía a tu aplicación: un balanceador, Nginx, un API Gateway.

-----

## El problema

La API de la tienda no tiene ningún límite. En una sola semana:

* un script prueba 40 000 contraseñas contra `/api/v1/auth/login` en una hora: **fuerza bruta**;
* un competidor descarga el catálogo completo con precios cada cinco minutos: **scraping**;
* un cliente con un bug en su integración llama a `/api/v1/orders` en un bucle infinito, 300 veces por segundo, y la base de datos se satura para **todos** los usuarios;
* el endpoint `/api/v1/reports/export`, que tarda 20 segundos, recibe 50 pedidos simultáneos y agota la memoria del servidor.

Las consecuencias: la disponibilidad depende del cliente que peor se porte, y el login no tiene ninguna defensa contra la fuerza bruta.

-----

## Cómo funciona

### 1. Para qué sirve el rate limiting

| Objetivo | Ejemplo |
| --- | --- |
| **Proteger la disponibilidad** | Que un cliente con un bug no sature la base de datos para todos |
| **Frenar ataques** | Fuerza bruta en el login, credential stuffing, scraping |
| **Repartir los recursos de forma justa** | Ningún cliente acapara la capacidad |
| **Controlar costos** | Limitar llamadas a endpoints caros (reportes, IA, APIs externas pagas) |
| **Planes comerciales** | El plan gratuito permite 100 peticiones por minuto; el pago, 10 000 |

Es una defensa de la **disponibilidad** (la "A" de la tríada CIA), y aparece en el OWASP API Security Top 10 como *Unrestricted Resource Consumption*.

### 2. Los algoritmos

| Algoritmo | Cómo funciona | Ventaja | Desventaja |
| --- | --- | --- | --- |
| **Ventana fija** | 100 peticiones por minuto, contadas de 10:00:00 a 10:00:59 | Simple, poca memoria | Permite ráfagas en el borde: 100 a las 10:00:59 y 100 a las 10:01:00 |
| **Ventana deslizante** | 100 peticiones en los últimos 60 segundos, divididos en segmentos | Reparte mejor, sin el problema del borde | Algo más de memoria y cálculo |
| **Token bucket** | Un balde de 20 tokens que se recarga a 1 por segundo; cada petición usa uno | Permite ráfagas cortas y controla el promedio | Un poco más difícil de explicar a los clientes |
| **Concurrencia** | Máximo 5 peticiones **en proceso** a la vez | Ideal para operaciones lentas o pesadas | No limita la cantidad total en el tiempo |

```text
Ventana fija (100/min):
  10:00:00 ─────────────── 10:00:59 │ 10:01:00 ─────────────── 10:01:59
             ...100 peticiones al final ││ 100 al principio...
                                  ↑ 200 peticiones en 2 segundos

Token bucket (capacidad 20, recarga 1/s):
  ráfaga de 20 → balde vacío → 1 petición por segundo → si el cliente para, se vuelve a llenar
```

### 3. Rate limiting en ASP.NET Core

Desde .NET 7, el middleware viene integrado en `Microsoft.AspNetCore.RateLimiting`:

```csharp
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    o.AddFixedWindowLimiter("fijo", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueLimit = 0;
    });

    o.AddSlidingWindowLimiter("deslizante", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 6; // segmentos de 10 segundos
    });

    o.AddTokenBucketLimiter("rafagas", opt =>
    {
        opt.TokenLimit = 20;
        opt.TokensPerPeriod = 5;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(1);
    });

    o.AddConcurrencyLimiter("reportes", opt =>
    {
        opt.PermitLimit = 3;
        opt.QueueLimit = 5; // hasta 5 esperan su turno; el resto, 429
    });
});

var app = builder.Build();
app.UseRateLimiter();
```

Se aplica por endpoint:

```csharp
[EnableRateLimiting("reportes")]
[HttpPost("api/v1/reports/export")]
public Task<IActionResult> Exportar() { /* ... */ }

app.MapGet("/api/v1/products", ListarProductos).RequireRateLimiting("deslizante");
```

> **Ojo:** el código de rechazo por defecto del middleware es **`503 Service Unavailable`**, no `429`. Si no configuras `RejectionStatusCode`, los clientes van a interpretar que tu servidor está caído, y sus reintentos automáticos (que reintentan `503`) empeoran la situación.

> **Ojo con el orden:** `UseRateLimiter()` va **después** de `UseRouting()` (si lo usas explícitamente) para que funcione `[EnableRateLimiting]`, y después de `UseAuthentication()` si la partición usa la identidad del usuario.

### 4. Particionar: ¿límite para quién?

Los limitadores con nombre de la sección anterior tienen **un solo contador para todos**: con 100 por minuto, entre **todos** los usuarios no pueden hacer más de 100. Casi siempre lo que quieres es un límite **por cliente**. Para eso se usa un limitador particionado:

```csharp
o.AddPolicy("por-usuario", ctx =>
{
    var clave = ctx.User.Identity?.IsAuthenticated == true
        ? $"usuario:{ctx.User.FindFirstValue(ClaimTypes.NameIdentifier)}"
        : $"ip:{ctx.Connection.RemoteIpAddress}";

    return RateLimitPartition.GetSlidingWindowLimiter(clave, _ => new SlidingWindowRateLimiterOptions
    {
        PermitLimit = 100,
        Window = TimeSpan.FromMinutes(1),
        SegmentsPerWindow = 6
    });
});
```

| Criterio de partición | Ventaja | Problema |
| --- | --- | --- |
| **Usuario autenticado** | Justo y preciso | No sirve para endpoints anónimos (como el login) |
| **IP** | Funciona sin autenticación | Muchos usuarios legítimos pueden compartir una IP (una oficina, una universidad, una red móvil); un atacante puede usar muchas IPs |
| **API key** | Ideal para integraciones entre sistemas y planes comerciales | Solo aplica a clientes con clave |
| **Combinado** | Usuario si está autenticado, IP si no | Es la opción más habitual |

Para el login, conviene combinar dos límites: por **IP** (frena a un atacante probando muchas cuentas) y por **cuenta** (frena muchos intentos contra una misma cuenta, aunque vengan de IPs distintas). Este último es el bloqueo por intentos de [Autenticación](05-autenticacion-y-datos-sensibles.md).

> **Ojo con los proxies:** si tu aplicación está detrás de un balanceador, un proxy inverso o un CDN, `RemoteIpAddress` es la IP **del proxy**, no la del cliente: todos los usuarios comparten un solo contador. Hay que configurar `UseForwardedHeaders()` para leer el header `X-Forwarded-For`, **confiando solo en los proxies conocidos**: si aceptas ese header de cualquiera, un atacante lo falsifica y evade el límite.

### 5. Responder bien: `429` con `Retry-After`

Cuando se rechaza una petición, el cliente necesita saber **cuándo** puede volver a intentar:

```csharp
o.OnRejected = async (context, ct) =>
{
    if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var espera))
        context.HttpContext.Response.Headers.RetryAfter = ((int)espera.TotalSeconds).ToString();

    context.HttpContext.RequestServices.GetRequiredService<ILogger<Program>>()
        .LogWarning("Rate limit superado: {Ruta} desde {Ip}",
            context.HttpContext.Request.Path, context.HttpContext.Connection.RemoteIpAddress);

    await context.HttpContext.RequestServices.GetRequiredService<IProblemDetailsService>()
        .WriteAsync(new ProblemDetailsContext
        {
            HttpContext = context.HttpContext,
            ProblemDetails = { Title = "Demasiadas peticiones", Status = StatusCodes.Status429TooManyRequests }
        });
};
```

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 23
Content-Type: application/problem+json

{ "title": "Demasiadas peticiones", "status": 429 }
```

Del otro lado, un cliente bien hecho (por ejemplo, el handler de resiliencia de [la lección anterior](07-resiliencia-con-polly.md)) respeta el `Retry-After` en lugar de reintentar de inmediato.

Los rechazos también son **eventos de seguridad**: muchos `429` en el login desde una IP son una señal de ataque que conviene alertar (OWASP A09).

### 6. Varias instancias: el contador vive en memoria

El rate limiter integrado guarda los contadores **en la memoria de cada instancia**. Si la API corre en 3 instancias detrás de un balanceador, un cliente puede hacer 100 peticiones por minuto **en cada una**: 300 en total.

| Opción | Cómo |
| --- | --- |
| Aceptar la aproximación | Dividir el límite por la cantidad de instancias (si el balanceo es parejo) |
| Contador distribuido | Guardar los contadores en Redis (librerías de terceros implementan los algoritmos sobre Redis) |
| Limitar antes de la aplicación | En el API Gateway, el CDN o el WAF (Azure API Management, AWS API Gateway, Cloudflare, Nginx) |

En la práctica, es habitual combinar: un límite global y grueso en el gateway (protección contra abuso masivo) y límites finos por usuario y por endpoint en la aplicación.

-----

## Ejemplo completo

La API del problema, con límites por tipo de endpoint.

`Program.cs`:

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    o.KnownProxies.Add(IPAddress.Parse("10.0.0.10")); // solo se confía en el balanceador
});

builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.OnRejected = EscribirRespuesta429; // Retry-After + Problem Details + log, como en la sección 5

    // general: 100 por minuto por usuario, o por IP si es anónimo
    o.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
        RateLimitPartition.GetSlidingWindowLimiter(ClaveCliente(ctx), _ => new SlidingWindowRateLimiterOptions
        {
            PermitLimit = 100,
            Window = TimeSpan.FromMinutes(1),
            SegmentsPerWindow = 6
        }));

    // login: 5 intentos por minuto por IP
    o.AddPolicy("login", ctx =>
        RateLimitPartition.GetFixedWindowLimiter($"login:{ctx.Connection.RemoteIpAddress}", _ =>
            new FixedWindowRateLimiterOptions { PermitLimit = 5, Window = TimeSpan.FromMinutes(1) }));

    // exportaciones: máximo 2 en proceso por usuario
    o.AddPolicy("reportes", ctx =>
        RateLimitPartition.GetConcurrencyLimiter(ClaveCliente(ctx), _ =>
            new ConcurrencyLimiterOptions { PermitLimit = 2, QueueLimit = 0 }));
});

var app = builder.Build();
app.UseForwardedHeaders();
app.UseAuthentication();
app.UseRateLimiter();
app.UseAuthorization();
app.MapControllers();
app.Run();

static string ClaveCliente(HttpContext ctx) =>
    ctx.User.Identity?.IsAuthenticated == true
        ? $"usuario:{ctx.User.FindFirstValue(ClaimTypes.NameIdentifier)}"
        : $"ip:{ctx.Connection.RemoteIpAddress}";
```

En los controllers:

```csharp
[EnableRateLimiting("login")]
[HttpPost("api/v1/auth/login")]
public Task<IActionResult> Login(LoginRequest req) { /* ... */ }

[EnableRateLimiting("reportes")]
[HttpPost("api/v1/reports/export")]
public Task<IActionResult> Exportar(ExportRequest req) { /* ... */ }
```

Qué pasa ahora con cada caso del problema:

| Caso | Límite que lo frena | Resultado |
| --- | --- | --- |
| Fuerza bruta en el login | 5 por minuto por IP + bloqueo de cuenta tras 5 fallos | 40 000 intentos pasan a tardar meses |
| Scraping del catálogo | 100 por minuto por IP | El catálogo completo tarda horas, no minutos |
| Integración con un bug en bucle | 100 por minuto por usuario | Solo ese cliente recibe `429`; los demás no lo notan |
| 50 exportaciones simultáneas | 2 concurrentes por usuario | La memoria del servidor queda protegida |

Qué observar:

* Hay un límite global para todo y políticas más estrictas para los endpoints sensibles.
* El limitador global y la política de un endpoint se suman: una petición al login tiene que pasar los dos.
* La IP real se lee del header del balanceador, pero solo si viene de ese balanceador.
* Cada rechazo responde `429` con `Retry-After` y se registra, para poder alertar ante ataques.

-----

## Errores comunes

**1. Dejar el código de rechazo por defecto.**
Qué pasa: los clientes reciben `503` y creen que el servidor está caído; sus reintentos aumentan la carga.
Por qué: el middleware de ASP.NET Core rechaza con `503` si no se configura.
Arreglo: `RejectionStatusCode = 429` y `Retry-After`.

**2. Un límite global sin particionar.**
Qué pasa: un solo cliente consume todo el cupo y bloquea a los demás.
Por qué: `AddFixedWindowLimiter` con nombre usa un único contador compartido.
Arreglo: limitadores particionados por usuario, IP o API key.

**3. Leer la IP del proxy.**
Qué pasa: todos los usuarios comparten un contador (o nadie es limitado).
Por qué: detrás de un balanceador, `RemoteIpAddress` es la IP del balanceador.
Arreglo: `UseForwardedHeaders()` con los proxies conocidos configurados.

**4. Confiar en `X-Forwarded-For` de cualquiera.**
Qué pasa: un atacante envía un header falso distinto en cada petición y evade el límite.
Por qué: se acepta el header sin verificar que venga de un proxy propio.
Arreglo: `KnownProxies` o `KnownNetworks` con solo tu infraestructura.

**5. Límites en memoria con muchas instancias, sin tenerlo en cuenta.**
Qué pasa: el límite real es el configurado multiplicado por la cantidad de instancias.
Por qué: cada instancia tiene sus propios contadores.
Arreglo: límite en el gateway, contador distribuido o ajustar el límite por instancia.

**6. Solo rate limiting por IP en el login.**
Qué pasa: un atacante con muchas IPs (una botnet) prueba contraseñas contra una cuenta sin ser frenado.
Por qué: se limita al atacante, pero no se protege a la cuenta.
Arreglo: combinar el límite por IP con el bloqueo por cuenta.

-----

## Evolución y versiones

* **Antes de .NET 7:** se usaban librerías de terceros, como AspNetCoreRateLimit, o se limitaba en el proxy.
* **.NET 7 (2022):** `System.Threading.RateLimiting` y el middleware `Microsoft.AspNetCore.RateLimiting` integrados, con ventana fija, deslizante, token bucket y concurrencia.
* **2023:** el OWASP API Security Top 10 incluye *Unrestricted Resource Consumption* (API4).

-----

## Cuándo sí y cuándo no

**Usa rate limiting siempre en:** login, registro, recuperación de contraseña, envío de códigos o emails, endpoints costosos y cualquier API pública.

**Usa límite de concurrencia cuando:** el endpoint es lento o pesado (reportes, exportaciones, procesamiento de archivos).

**Usa límites en el gateway cuando:** tienes varias instancias, varios servicios, o necesitas frenar el abuso antes de que llegue a la aplicación.

**No uses límites demasiado bajos** en endpoints que la propia interfaz llama muchas veces (autocompletado, paginación): vas a bloquear a tus usuarios legítimos. Mide el uso normal antes de fijar los valores.

-----

## Resumen en 5 líneas

1. El rate limiting protege la disponibilidad, frena la fuerza bruta y el scraping, y reparte los recursos de forma justa.
2. Algoritmos: ventana fija (simple), deslizante (sin ráfagas en el borde), token bucket (ráfagas controladas) y concurrencia (operaciones pesadas).
3. En ASP.NET Core: `AddRateLimiter` + `UseRateLimiter`, con `RejectionStatusCode = 429`, porque el valor por defecto es `503`.
4. Particiona por usuario, IP o API key; detrás de un proxy, `UseForwardedHeaders` solo con proxies conocidos.
5. Los contadores viven en memoria por instancia: para varias instancias, gateway o contador distribuido; responde siempre con `Retry-After`.

-----

## Para profundizar

<details>
<summary>Headers informativos de límites</summary>

Muchas APIs públicas informan el estado del límite en cada respuesta, para que los clientes se regulen solos antes de recibir un `429`:

```http
RateLimit-Limit: 100
RateLimit-Remaining: 37
RateLimit-Reset: 23
```

Hay un borrador de estándar del IETF para estos headers (`RateLimit` y `RateLimit-Policy`); GitHub y otras APIs usan variantes con el prefijo `X-RateLimit-`. ASP.NET Core no los agrega solo: se pueden escribir en un middleware a partir de las estadísticas del limitador.

</details>

<details>
<summary>Rate limiting frente a cuotas y planes</summary>

El rate limiting protege la infraestructura en el corto plazo (segundos y minutos). Las **cuotas** controlan el uso comercial en el largo plazo (10 000 llamadas por mes en el plan gratuito). Las cuotas necesitan almacenamiento persistente (una base de datos o Redis), facturación y avisos al cliente cuando se acerca al límite, y normalmente se implementan en un API Gateway con gestión de suscripciones, como Azure API Management.

</details>

-----

## En entrevista

### Respuesta corta (junior)

El rate limiting limita cuántas peticiones puede hacer un cliente en un tiempo determinado, por ejemplo 100 por minuto. Sirve para proteger la API de abusos y ataques de fuerza bruta. Cuando se supera, se responde `429 Too Many Requests`.

### Respuesta ampliada (semi-senior)

Lo uso para proteger la disponibilidad y frenar ataques. En ASP.NET Core configuro el middleware integrado con `RejectionStatusCode = 429`, porque el valor por defecto es `503`, y respondo con `Retry-After` y Problem Details. Elijo el algoritmo según el caso: ventana deslizante para el límite general, ventana fija estricta para el login y límite de concurrencia para endpoints pesados como exportaciones. Particiono por usuario autenticado y por IP para anónimos, y detrás de un balanceador configuro los forwarded headers confiando solo en los proxies conocidos. Sé que los contadores viven en memoria por instancia, así que con varias instancias combino un límite grueso en el gateway con límites finos en la aplicación. Y en el login lo combino con el bloqueo por cuenta, porque el límite por IP no frena a una botnet.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre ventana fija y token bucket?**
La ventana fija cuenta peticiones en intervalos fijos y permite ráfagas en el borde entre dos ventanas. El token bucket se recarga a ritmo constante: permite ráfagas cortas hasta la capacidad del balde y controla el promedio.

**2. ¿Por qué no alcanza con limitar por IP?**
Porque muchos usuarios legítimos pueden compartir una IP, y un atacante con muchas IPs evade el límite. Conviene combinarlo con límites por usuario o por cuenta.

**3. ¿Qué pasa con el rate limiting si escalas a varias instancias?**
Cada instancia cuenta por separado, así que el límite real se multiplica. Se resuelve con un límite en el gateway o con contadores compartidos en Redis.

-----

## Práctica

**Ejercicio 1.** Elige el algoritmo y la partición para cada endpoint.

1. `POST /auth/login`
2. `GET /products` (catálogo público)
3. `POST /reports/export` (tarda 30 segundos)
4. `POST /sms/verification-code` (cada SMS cuesta dinero)
5. API para integraciones de clientes con plan gratuito y plan pago

<details>
<summary>Solución</summary>

1. **Ventana fija estricta por IP** (5 por minuto), más bloqueo por cuenta.
2. **Ventana deslizante por IP** con un límite generoso (por ejemplo, 300 por minuto), y caché o CDN para aliviar la carga.
3. **Concurrencia por usuario** (1 o 2 en proceso).
4. **Ventana fija por número de teléfono y por usuario** (3 por hora), además del límite por IP. Es un caso típico de abuso de flujos de negocio (API6).
5. **Token bucket por API key**, con límites distintos según el plan, idealmente gestionado en un API Gateway con cuotas.

</details>

**Ejercicio 2.** ¿Qué problemas tiene esta configuración? La API corre en 4 instancias detrás de un balanceador.

```csharp
builder.Services.AddRateLimiter(o =>
    o.AddFixedWindowLimiter("api", opt => { opt.PermitLimit = 1000; opt.Window = TimeSpan.FromMinutes(1); }));

app.UseRateLimiter();
app.MapControllers().RequireRateLimiting("api");
```

<details>
<summary>Solución</summary>

* **Sin `RejectionStatusCode`:** rechaza con `503`.
* **Un solo contador para todos:** 1000 por minuto **entre todos los clientes**; un solo cliente puede consumirlo entero.
* **Sin `UseForwardedHeaders`:** si se particionara por IP, todos compartirían la IP del balanceador.
* **4 instancias:** cada una cuenta por separado; el límite efectivo es de unas 4000 por minuto.
* **Sin `Retry-After`:** los clientes no saben cuándo reintentar.

Corrección: limitador particionado por usuario o IP con `RejectionStatusCode = 429`, `OnRejected` con `Retry-After`, `UseForwardedHeaders` con los proxies conocidos, y el límite ajustado por instancia o movido al gateway.

</details>

-----

## Siguiente lección

Terminaste las lecciones de esta fase. Para continuar con el roadmap: [Bases de datos y rendimiento](../../README.md) (punto 6).
