# Interpretación de logs

## En una frase

Un log útil es un **evento estructurado con campos filtrables y un correlation ID**: el nivel te dice que algo falló, y el correlation ID te deja reconstruir la secuencia completa de esa transacción para entender por qué.

-----

## Antes de empezar

Conviene que ya sepas:

* Cuándo sirve el depurador y cuándo no: [Depuración avanzada](02-depuracion-avanzada.md).
* Inyección de dependencias en ASP.NET Core, aunque sea de forma básica.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Log estructurado:** registro con campos con nombre (`UserId`, `PedidoId`) en lugar de solo texto libre.
* **Plantilla de mensaje (*message template*):** texto con placeholders con nombre, como `"Login fallido para {UserId}"`.
* **Nivel de log (*log level*):** severidad de un registro, de `Trace` a `Critical`.
* **Correlation ID:** identificador único que acompaña a una transacción a través de todos los servicios que toca.
* **Sink:** destino al que Serilog envía los logs: consola, archivo, base de datos, Seq.
* **Categoría:** nombre que identifica el origen de un log; con `ILogger<T>` es el nombre completo de `T`.
* **Downstream:** servicios a los que llama el sistema actual, más abajo en la cadena de una petición.

-----

## El problema

A las 03:00 falla el cobro de un pedido. A la mañana, alguien abre el archivo de logs:

```text
2026-10-24 03:12:01 El usuario 8812 inició el checkout
2026-10-24 03:12:01 El usuario 4410 inició el checkout
2026-10-24 03:12:02 Validando stock
2026-10-24 03:12:02 Validando stock
2026-10-24 03:12:02 Stock bajo
2026-10-24 03:12:03 Iniciando cobro
2026-10-24 03:12:03 Error al procesar pago
2026-10-24 03:12:03 Iniciando cobro
... 2 millones de líneas más
```

Las consecuencias:

* no se sabe a qué usuario ni a qué pedido corresponde cada línea, porque las peticiones concurrentes se mezclan;
* para buscar, solo queda hacer un grep de texto sobre millones de líneas;
* el mensaje "Error al procesar pago" no dice nada de la causa, y no hay forma de saber qué pasó antes en **esa** transacción.

-----

## Cómo funciona

### 1. Logging estructurado frente a texto plano

En lugar de interpolar el texto:

```csharp
_logger.LogInformation($"El usuario {userId} falló el login");
```

usa una plantilla con placeholders con nombre y pasa los valores como argumentos:

```csharp
_logger.LogInformation("Login fallido para {UserId}, razón: {Reason}", userId, reason);
```

La diferencia no es cosmética. Con la plantilla, `UserId` y `Reason` se guardan como **campos separados** (típicamente en JSON), no como texto embebido:

```json
{
  "@t": "2026-10-24T03:12:01Z",
  "@mt": "Login fallido para {UserId}, razón: {Reason}",
  "UserId": 8812,
  "Reason": "PasswordIncorrecta"
}
```

Eso te permite ir a Seq, ELK o Application Insights y filtrar `UserId = 8812` en milisegundos, en lugar de hacer grep sobre millones de líneas.

Si lo usas mal:

```csharp
_logger.LogInformation($"Login fallido para {userId}");
```

* El valor queda pegado al texto: no hay campo `UserId` para filtrar.
* Cada mensaje es una plantilla distinta, así que no puedes agrupar "todos los logins fallidos".
* El string se construye **siempre**, aunque el nivel `Information` esté desactivado.

Los analizadores de .NET lo marcan con la advertencia **CA2254** ("la plantilla debe ser una expresión estática").

### 2. Niveles de log

`ILogger` (de `Microsoft.Extensions.Logging`) y Serilog usan nombres distintos para niveles equivalentes:

| Severidad | Valor | `ILogger` | Serilog |
| --- | --- | --- | --- |
| Más detallado | 0 | `Trace` | `Verbose` |
| Diagnóstico | 1 | `Debug` | `Debug` |
| Flujo normal | 2 | `Information` | `Information` |
| Anómalo, pero recuperable | 3 | `Warning` | `Warning` |
| Falló una operación | 4 | `Error` | `Error` |
| Falló el sistema | 5 | `Critical` | `Fatal` |

El valor numérico importa porque así funciona el filtro: el nivel configurado es un **piso**. Deja pasar ese nivel y todos los mayores, y descarta los menores.

Cuando Serilog se usa detrás de `ILogger` (el patrón típico en ASP.NET Core), mapea `LogLevel.Critical` a `Fatal`. En el código escribes `_logger.LogCritical(...)`, pero en la configuración de Serilog vas a ver `"Fatal"`. No es un error, es el mapeo esperado: no busques `LogLevel.Fatal` en el código, porque no existe.

### 3. Correlation ID

En un sistema distribuido (frontend → API → otro microservicio → base de datos), una sola petición del usuario puede tocar 4 o 5 sistemas. Si algo falla, ¿cómo encuentras los logs de **esa** transacción en todos ellos?

Con un **correlation ID**: un identificador único que se genera al entrar la petición y viaja en cada llamada downstream como header HTTP (por ejemplo, `X-Correlation-Id`). Cada log escrito durante esa petición lo incluye como campo.

En ASP.NET Core se implementa con un middleware que:

1. Lee el header `X-Correlation-Id` si viene o genera uno nuevo (`Guid.NewGuid()`) si no.
2. Lo agrega al `LogContext` de Serilog (`LogContext.PushProperty("CorrelationId", id)`), para que todos los logs de esa petición lo incluyan.
3. Lo propaga en el header de las llamadas HTTP salientes.

```csharp
public class CorrelationIdMiddleware(RequestDelegate next)
{
    private const string Header = "X-Correlation-Id";

    public async Task InvokeAsync(HttpContext context)
    {
        string correlationId = context.Request.Headers.TryGetValue(Header, out var valor)
            ? valor.ToString()
            : Guid.NewGuid().ToString();

        context.Response.Headers[Header] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await next(context);
        }
    }
}
```

> **Ojo:** `LogContext.PushProperty` no hace nada si Serilog no está configurado con `.Enrich.FromLogContext()`. Es el motivo más común de "agregué el correlation ID y no aparece en los logs".

### 4. Cómo leer los logs ante un error

Encontrar un log de `Error` no es el final de la investigación, es el punto de partida.

| Paso | Qué haces | Qué obtienes |
| --- | --- | --- |
| 1 | Filtrar `Level >= Error` | La **alarma**: qué falló |
| 2 | Copiar el correlation ID de ese log | La llave de esa transacción |
| 3 | Filtrar por ese correlation ID, **sin** filtrar por nivel | Todos los logs de esa transacción, también los `Information` |
| 4 | Ordenar por timestamp y leer de arriba hacia abajo | La **historia**: qué se ejecutó y qué pasó justo antes del error |

**Regla práctica: filtrar por nivel te da la alarma; filtrar por correlation ID te da la causa.**

### 5. Inyección de `ILogger<T>`

`ILogger` se inyecta como cualquier otro servicio:

```csharp
public class PedidoController(ILogger<PedidoController> logger) : ControllerBase
{
    // ...
}
```

El tipo genérico no es decorativo: define la **categoría** del log (el nombre completo de la clase, como `MiApp.Controllers.PedidoController`). Eso permite configurar el nivel por clase o por namespace, no solo de forma global.

### 6. Configuración de niveles en `appsettings.json`

El nivel no se controla en el código, se controla en la configuración. Así puedes bajar el piso a `Debug` en un ambiente puntual sin recompilar.

```json
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "Microsoft.AspNetCore": "Warning"
  }
}
```

* `Default` define el nivel mínimo para toda la aplicación.
* Cualquier otra clave (como `Microsoft.AspNetCore`) lo sobrescribe para ese namespace, típicamente para reducir el ruido del framework.

Con `Default: Information`, `Trace` y `Debug` quedan filtrados. Por eso en producción no ves logs de `Debug`: existen en el código, pero la configuración los descarta.

> **Ojo:** si usas Serilog con `UseSerilog()`, los niveles se leen de la sección `"Serilog": { "MinimumLevel": ... }`, no de `"Logging"`. Cambiar `"Logging"` en ese caso no tiene ningún efecto.

### 7. Logs en base de datos

Guardar logs en una tabla (en lugar de, o además de, consola y archivos) es común en producción para tener trazabilidad histórica: responder "muéstrame todos los errores de ayer" con una consulta.

No se implementa a mano: se agrega un **sink** de base de datos a Serilog (por ejemplo, `Serilog.Sinks.MSSqlServer`) o a NLog. El código de negocio no cambia:

```csharp
_logger.LogError(ex, "Error al crear el pedido {PedidoId}", pedidoId);
```

La configuración decide si ese log va a consola, a un archivo, a una tabla SQL o a los tres destinos.

| Campo | Para qué sirve |
| --- | --- |
| `Id` | Identificador |
| `Level` | Information, Error, etc. |
| `Message` | El mensaje ya formateado |
| `Timestamp` | Cuándo ocurrió |
| `Source` | Controller o servicio que lo generó |
| `Exception` | El error completo, si lo hay |

**Regla práctica: no guardes todo en la base.** Los errores y los eventos de negocio importantes sí van a la tabla. Los logs de `Debug` o de alto volumen no: llenan la base y degradan el rendimiento sin aportar valor a largo plazo. Lo habitual es archivo o consola para todo, y base de datos solo para lo que necesita auditoría o búsqueda estructurada.

> **En la práctica:** en los trabajos donde apliqué este patrón, los errores y los fallos de procesos de negocio efectivamente terminaban en una tabla. Es lo que vas a encontrar en la mayoría de los equipos que usan .NET en producción.

> **Ojo:** Seq no es una librería de logging como Serilog o NLog: es un servidor para consultar logs estructurados. Serilog escribe, y Seq es uno de los destinos posibles, igual que una base de datos.

-----

## Ejemplo completo

El mismo fallo de las 03:00, con logs estructurados y correlation ID.

`Program.cs`:

```csharp
using Serilog;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft.AspNetCore", Serilog.Events.LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .WriteTo.Console(new Serilog.Formatting.Compact.CompactJsonFormatter())
    .WriteTo.Seq("http://localhost:5341")
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog();
builder.Services.AddControllers();

var app = builder.Build();
app.UseMiddleware<CorrelationIdMiddleware>();
app.MapControllers();
app.Run();
```

Paquetes: `Serilog.AspNetCore` y `Serilog.Sinks.Seq`.

`PedidoService.cs`:

```csharp
public class PedidoService(ILogger<PedidoService> logger, IPasarelaPagos pasarela)
{
    public async Task ProcesarAsync(Pedido pedido)
    {
        logger.LogInformation("Pedido {PedidoId} recibido", pedido.Id);
        logger.LogInformation("Validando stock de {CantidadProductos} productos", pedido.Lineas.Count);

        foreach (var linea in pedido.Lineas.Where(l => l.StockDisponible < 5))
        {
            logger.LogWarning("Producto {Sku} con stock bajo ({Stock} unidades)", linea.Sku, linea.StockDisponible);
        }

        logger.LogInformation("Iniciando cobro con {Proveedor}", pasarela.Nombre);
        try
        {
            await pasarela.CobrarAsync(pedido);
        }
        catch (HttpRequestException ex)
        {
            logger.LogError(ex, "No se pudo procesar el pago del pedido {PedidoId}", pedido.Id);
            throw;
        }
    }
}
```

**Paso 1.** En Seq filtras `@Level = 'Error'`:

```text
[Error] No se pudo procesar el pago del pedido 4521   CorrelationId = a1b2c3d4
```

**Pasos 2 y 3.** Filtras `CorrelationId = 'a1b2c3d4'`, sin filtrar por nivel:

```text
03:12:01 [Information] Pedido 4521 recibido
03:12:01 [Information] Validando stock de 3 productos
03:12:02 [Warning]     Producto SKU-892 con stock bajo (2 unidades)
03:12:02 [Information] Iniciando cobro con PagosYa
03:12:33 [Error]       No se pudo procesar el pago del pedido 4521
                       System.Net.Http.HttpRequestException: ... (503) Service Unavailable
```

**Paso 4.** Leyendo la secuencia:

* El stock solo generó un `Warning` y no bloqueó el flujo: no es la causa.
* Entre "Iniciando cobro" y el error pasaron **31 segundos**, y la excepción es un 503 de PagosYa.
* La causa probable está en el proveedor de pagos, no en tu código.

Qué observar:

* El log de error solo decía "no se pudo procesar el pago". La causa apareció en la **secuencia** y en los timestamps.
* Todos los placeholders tienen nombre (`{PedidoId}`, `{Sku}`), así que puedes buscar "todos los pedidos con stock bajo de SKU-892" sin grep.
* `LogError(ex, ...)` recibe la excepción como primer argumento: así el stack trace completo queda en el campo `Exception`, no perdido en el texto.

-----

## Errores comunes

**1. Interpolar strings en los logs.**
Qué pasa: no puedes filtrar por campos ni agrupar mensajes iguales; aparece la advertencia CA2254.
Por qué: con `$"..."` el valor se pega al texto antes de llegar al logger.
Arreglo: plantilla con placeholders con nombre y valores como argumentos.

**2. El correlation ID no aparece en los logs.**
Qué pasa: el middleware está, pero el campo `CorrelationId` no se ve en ningún log.
Por qué: falta `.Enrich.FromLogContext()` en la configuración de Serilog.
Arreglo: agrégalo a la `LoggerConfiguration`.

**3. Cambiar el nivel en la sección equivocada.**
Qué pasa: cambias `"Logging": { "LogLevel": ... }` y no cambia nada.
Por qué: con `UseSerilog()`, Serilog lee su propia sección `"Serilog"`.
Arreglo: configura `"Serilog": { "MinimumLevel": ... }` o hazlo en código con `.MinimumLevel`.

**4. Loguear la excepción como texto.**
Qué pasa: el log dice `Error: Object reference not set...` y no hay stack trace.
Por qué: `logger.LogError("Error: " + ex.Message)` descarta todo menos el mensaje.
Arreglo: `logger.LogError(ex, "Error al crear el pedido {PedidoId}", pedidoId)`.

**5. Investigar filtrando solo por nivel.**
Qué pasa: ves el error, pero no qué pasó antes.
Por qué: los logs que explican la causa suelen ser `Information` o `Warning`.
Arreglo: usa el error solo para obtener el correlation ID y después filtra por ese ID.

**6. Loguear datos sensibles.**
Qué pasa: contraseñas, tokens o números de tarjeta quedan guardados en texto plano en el sistema de logs.
Por qué: se loguea el objeto completo (`{@Request}`) sin pensar qué contiene.
Arreglo: loguea solo los campos necesarios y nunca credenciales ni datos de pago.

-----

## Cuándo sí y cuándo no

**Usa logs estructurados cuando:** el sistema corre en un servidor o en producción, atiende peticiones concurrentes o involucra más de un servicio. Es decir, casi siempre fuera de tu máquina.

**Usa el depurador en lugar de logs cuando:** puedes reproducir el bug localmente y necesitas ver el estado completo en un punto concreto.

**No loguees cuando:** el dato es sensible, el volumen es tan alto que el costo de almacenarlo supera su valor (por ejemplo, un log por cada elemento de un bucle de un millón) o la información ya está en otra herramienta, como métricas o trazas.

-----

## Resumen en 5 líneas

1. Loguea con plantillas y placeholders con nombre, nunca con interpolación: los campos se vuelven filtrables.
2. El nivel configurado es un piso; `Critical` en `ILogger` equivale a `Fatal` en Serilog.
3. El correlation ID viaja en cada petición y permite reconstruir una transacción completa; con Serilog requiere `Enrich.FromLogContext()`.
4. Filtrar por nivel te da la alarma; filtrar por correlation ID te da la causa.
5. Los niveles se configuran fuera del código, y a la base de datos solo va lo que necesita auditoría.

-----

## Para profundizar

<details>
<summary>Correlation ID frente a trazas distribuidas (OpenTelemetry)</summary>

ASP.NET Core ya propaga un identificador de traza estándar: el header W3C `traceparent`, expuesto en el código como `Activity.Current.TraceId`. `HttpClient` lo envía automáticamente a los servicios downstream. Con **OpenTelemetry** obtienes, además de logs correlacionados, la duración de cada tramo (*span*) de la petición en cada servicio.

El `X-Correlation-Id` manual sigue siendo útil cuando hay sistemas que no hablan W3C (colas, sistemas legados) o cuando quieres un ID legible para el área de soporte. Los dos enfoques pueden convivir.

</details>

<details>
<summary>Logs de alto rendimiento con <code>LoggerMessage</code></summary>

En rutas muy calientes, incluso la plantilla tiene costo: el *boxing* de los argumentos y el parseo de la plantilla. .NET 6 agregó un generador de código fuente:

```csharp
public static partial class Logs
{
    [LoggerMessage(Level = LogLevel.Warning, Message = "Producto {Sku} con stock bajo ({Stock} unidades)")]
    public static partial void StockBajo(this ILogger logger, string sku, int stock);
}

// uso
logger.StockBajo(linea.Sku, linea.StockDisponible);
```

El compilador genera el código optimizado y valida la plantilla en tiempo de compilación.

</details>

<details>
<summary>Scopes en <code>ILogger</code></summary>

Sin depender de Serilog, `ILogger` permite agregar campos a todos los logs de un bloque:

```csharp
using (logger.BeginScope(new Dictionary<string, object> { ["PedidoId"] = pedido.Id }))
{
    // todos los logs de aquí adentro incluyen PedidoId
}
```

Es el equivalente estándar de `LogContext.PushProperty`. Serilog los respeta si está configurado con `Enrich.FromLogContext()`.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Uso `ILogger` con Serilog y escribo los mensajes con placeholders con nombre en lugar de interpolar strings, para que los valores queden como campos que se pueden filtrar. Uso el nivel adecuado según la gravedad y configuro el nivel mínimo en `appsettings.json`.

### Respuesta ampliada (semi-senior)

Hago logging estructurado con plantillas, porque los campos con nombre permiten consultar los logs en Seq o Application Insights en lugar de hacer grep. Cada petición lleva un correlation ID, que genero o leo en un middleware y agrego al `LogContext`, y lo propago a los servicios downstream. Para investigar, uso el log de error solo para obtener ese ID y después leo la secuencia completa de la transacción, porque la causa suele estar en los `Information` o `Warning` previos. Los niveles los controlo por configuración y por namespace, y a la base de datos solo mando lo que necesita auditoría. Nunca logueo datos sensibles.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no usar interpolación de strings en los logs?**
Porque el valor se pega al texto y deja de ser un campo filtrable, cada mensaje se vuelve una plantilla distinta y el string se construye aunque el nivel esté desactivado.

**2. ¿Qué nivel usarías para un pago rechazado por fondos insuficientes?**
`Information` o `Warning`, no `Error`: es un resultado de negocio esperado, no una falla del sistema. `Error` es para cuando una operación falla por algo que no debería pasar, como que el proveedor responda 503.

**3. ¿Cómo seguirías una petición a través de tres microservicios?**
Con un correlation ID (o el `TraceId` de W3C) que viaje en los headers y quede como campo en los logs de los tres servicios, centralizados en una misma herramienta.

-----

## Práctica

**Ejercicio 1.** Tu configuración tiene `"Default": "Warning"` y `"MiApp.Pagos": "Debug"`. ¿Cuáles de estos logs se registran?

```csharp
// en MiApp.Pedidos.PedidoService
logger.LogInformation("Pedido creado");
logger.LogError("Pedido fallido");

// en MiApp.Pagos.PasarelaService
logger.LogDebug("Request a la pasarela");
logger.LogTrace("Headers enviados");
```

<details>
<summary>Solución</summary>

* `Pedido creado` (Information en `MiApp.Pedidos`): **no**, el piso es `Warning`.
* `Pedido fallido` (Error en `MiApp.Pedidos`): **sí**.
* `Request a la pasarela` (Debug en `MiApp.Pagos`): **sí**, ese namespace tiene piso `Debug`.
* `Headers enviados` (Trace en `MiApp.Pagos`): **no**, `Trace` (0) está por debajo de `Debug` (1).

</details>

**Ejercicio 2.** ¿Qué tiene de malo este log? Reescríbelo.

```csharp
catch (Exception ex)
{
    _logger.LogError($"Error con el usuario {usuario.Email} y password {usuario.Password}: {ex.Message}");
}
```

<details>
<summary>Solución</summary>

* Interpola el string: `Email` no queda como campo filtrable.
* Loguea la **contraseña**: es un problema de seguridad grave.
* Solo guarda `ex.Message`, así que se pierden el tipo de excepción y el stack trace.

```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "Error al autenticar al usuario {UsuarioId}", usuario.Id);
}
```

Se usa el `Id` en lugar del email para no exponer datos personales innecesarios, y la excepción completa se pasa como primer argumento.

</details>

-----

## Siguiente lección

[Stack traces](04-stack-traces.md)
