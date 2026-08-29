# Interpretación de Logs

## Logging estructurado vs texto plano

En vez de loguear texto libre con interpolación:

```csharp
_logger.LogInformation($"El usuario {userId} falló el login");
```

Loguear estructurado, con placeholders con nombre:

```csharp
_logger.LogInformation("Login fallido para {UserId}, razón: {Reason}", userId, reason);
```

La diferencia no es cosmética: con placeholders con nombre, Serilog guarda `UserId` y `Reason` como CAMPOS separados (típicamente en JSON), no como texto embebido. Eso te permite ir a Seq/ELK/Application Insights y filtrar por `UserId = 123` en milisegundos, en vez de hacer un grep de texto sobre millones de líneas.

Como ya implementaste Serilog, seguramente ya usas este patrón — el punto acá es el POR QUÉ: siempre placeholders con nombre, nunca interpolación de string (`$"..."`) en el mensaje de log.

## Niveles de log: la diferencia que importa

`ILogger` (interfaz estándar de `Microsoft.Extensions.Logging`) y Serilog usan nombres distintos para conceptos equivalentes:

| Severidad | Valor numérico | ILogger (`Microsoft.Extensions.Logging`) | Serilog |
|---|---|---|---|
| Más detallado | 0 | `Trace` | `Verbose` |
| Debug | 1 | `Debug` | `Debug` |
| Normal | 2 | `Information` | `Information` |
| Anómalo, recuperable | 3 | `Warning` | `Warning` |
| Falló un proceso | 4 | `Error` | `Error` |
| Cayó el sistema | 5 | `Critical` | `Fatal` |

El valor numérico importa porque así funciona internamente el filtro de nivel: cualquier configuración de log level actúa como un piso — deja pasar ese valor y todos los mayores, descarta los menores.

Como Serilog suele usarse como sink detrás de `ILogger` (patrón típico en ASP.NET Core), internamente MAPEA `LogLevel.Critical` de ILogger a `Fatal`. En el código C# escribes `_logger.LogCritical(...)`, pero en la configuración de Serilog (`appsettings.json`, filtros por nivel) vas a ver `"Fatal"`. No es un error — es el mapeo esperado — pero si no lo sabes, te confunde no encontrar `LogLevel.Fatal` en el código.

## Correlation ID

En un sistema distribuido (Frontend → API → otro microservicio → base de datos), un solo request de usuario puede tocar 4 o 5 sistemas distintos. Si algo falla, ¿cómo correlacionas los logs de todos esos sistemas para esa transacción puntual?

Respuesta: un **Correlation ID** — un identificador único que se genera al entrar el request (típicamente en un middleware) y viaja en cada llamada downstream como header HTTP (ej: `X-Correlation-Id`). Cada log que se escribe durante ese request incluye ese ID como campo estructurado.

En ASP.NET Core, esto se hace normalmente con un middleware que:

1. Lee el header `X-Correlation-Id` si viene, o genera uno nuevo (`Guid.NewGuid()`) si no.
2. Lo mete en el `LogContext` de Serilog (`LogContext.PushProperty("CorrelationId", id)`), para que TODOS los logs de ese request lo incluyan automáticamente.
3. Lo propaga en el header de las llamadas HTTP salientes a otros servicios.

Con eso, ante un error buscas el Correlation ID en Seq/ELK y ves el recorrido completo de esa transacción específica, en todos los servicios que tocó.

## Cómo leer e interpretar los logs

Cuando aparece un error, el orden de investigación importa. Encontrar UN log de `Error` no es el final de la historia — es el punto de partida.

**Paso 1: filtrar por nivel.** Arrancar filtrando `Level >= Error` (Error y Critical) separa el ruido normal (Information/Warning) de lo que realmente rompió algo. Esto da la alarma, pero rara vez el contexto completo.

**Paso 2: tomar el Correlation ID de ese log de error.** Cada log de esa misma transacción — sin importar su nivel — comparte ese mismo Correlation ID.

**Paso 3: filtrar TODOS los logs con ese Correlation ID, esta vez sin filtrar por nivel.** Ahora aparecen también los logs de `Information` que se generaron antes del error — la secuencia completa de lo que hizo esa transacción puntual, desde que entró el request hasta que falló.

**Paso 4: ordenar por Timestamp y leer de arriba hacia abajo.** Ahí se reconstruye la historia real: qué se ejecutó, en qué orden, y qué fue lo último que pasó antes del error — que muchas veces es más revelador que el mensaje de error en sí mismo.

**Ejemplo concreto:**

Un log de `Error` dice: *"No se pudo procesar el pago del pedido #4521"*. Ese mensaje solo, no explica el POR QUÉ. Se toma su Correlation ID (ej: `a1b2c3d4`), se filtra por ese ID, y aparece la secuencia completa:

```
[Information] a1b2c3d4 - Pedido #4521 recibido
[Information] a1b2c3d4 - Validando stock de 3 productos
[Warning]     a1b2c3d4 - Producto SKU-892 con stock bajo (2 unidades)
[Information] a1b2c3d4 - Iniciando cobro con proveedor de pagos
[Error]       a1b2c3d4 - No se pudo procesar el pago del pedido #4521
```

Recién con esta secuencia completa se entiende que el problema probablemente está relacionado con el proveedor de pagos, no con el stock (que solo generó un `Warning`, no bloqueó el flujo).

**Regla práctica: filtrar solo por nivel da la alarma; filtrar por Correlation ID da la causa.**

## Inyección de `ILogger`

`ILogger` es el servicio de logging estándar de .NET, y se inyecta por Dependency Injection igual que cualquier otro servicio:

```csharp
public class PedidoController : ControllerBase
{
    private readonly ILogger<PedidoController> _logger;

    public PedidoController(ILogger<PedidoController> logger)
    {
        _logger = logger;
    }
}
```

El tipo genérico (`PedidoController`) no es decorativo: define la **categoría** del log (internamente, `typeof(T).FullName`). Eso permite filtrar o configurar el nivel de logging por clase o por namespace completo, no solo de forma global.

## Configuración de niveles en `appsettings.json`

El nivel de log no se controla en el código — se controla en la configuración. Esto permite, por ejemplo, subir el nivel a `Debug` en un ambiente puntual sin tocar ni recompilar el código.

```json
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "Microsoft.AspNetCore": "Warning"
  }
}
```

- `Default` define el nivel mínimo para toda la aplicación.
- Cualquier clave adicional (como `Microsoft.AspNetCore`) sobreescribe ese default para ese namespace específico — típicamente para bajar el ruido que genera el propio framework.

Con `Default: Information`, por ejemplo, `Trace` y `Debug` quedan filtrados (no se registran) y `Information`, `Warning`, `Error` y `Critical` sí pasan. Por eso en producción normalmente no aparecen logs `Trace` ni `Debug`: no es que no existan en el código, es que están filtrados por configuración, no por código.

## Logging hacia base de datos

Guardar logs en una tabla (en vez de, o además de, consola/archivos) es una práctica común en producción para tener trazabilidad histórica: poder responder algo como "muéstrame todos los errores de ayer" con una query, en vez de grepear archivos de texto.

No se implementa a mano — se agrega un **sink** de base de datos a Serilog (o a NLog), que intercepta lo que ya se escribe con `ILogger` y lo persiste. El código de negocio no cambia:

```csharp
_logger.LogError(ex, "Error al crear el pedido {PedidoId}", pedidoId);
```

Serilog decide, según cómo esté configurado, si ese log va a consola, a un archivo, a una tabla SQL, o a las tres cosas a la vez — no se toca una línea de código de negocio para eso. (Ojo: Seq no es una librería de logging como Serilog o NLog — es un servidor/visor de logs estructurados. Se usa Serilog para escribir, y un sink de Seq como uno de los destinos posibles, igual que un sink de base de datos.)

Una tabla de logs típica:

| Campo | Para qué sirve |
|---|---|
| Id | Identificador |
| Level | Information, Error, etc. |
| Message | El mensaje del log |
| Timestamp | Cuándo ocurrió |
| Source | Controller o servicio que lo generó |
| Exception | El error completo, si lo hay |

**Regla práctica: no guardar todo en la base.** Logs críticos (errores, eventos de negocio importantes) sí van a la tabla. Logs de `Debug` o de alto volumen no — llenan la base y degradan el rendimiento sin aportar valor a largo plazo. Lo habitual es una combinación: archivo o consola para todo, base de datos solo para lo que necesita auditoría o búsqueda estructurada después.

> **En la práctica:** en los trabajos donde se aplicó este patrón, los logs críticos (errores, fallos de procesos de negocio) efectivamente terminan en una tabla de base de datos — coincide con la teoría. Es el patrón real que se va a encontrar en la mayoría de los equipos que usan .NET en producción.
