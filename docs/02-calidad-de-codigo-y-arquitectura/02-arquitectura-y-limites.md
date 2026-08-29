# Arquitectura y Límites entre Capas

## El principio central

No importa si el proyecto usa Clean Architecture, Onion Architecture o una simple Arquitectura en Capas — todas comparten la misma regla de fondo: **las dependencias apuntan hacia adentro, nunca hacia afuera.**

En la práctica, esto significa:

- El controlador/API **no debe contener lógica de negocio** — solo recibe la petición, la delega, y devuelve una respuesta.
- La capa de **Dominio no debe saber nada de la base de datos ni de frameworks externos** — no debería importar `Microsoft.EntityFrameworkCore` ni ninguna librería de infraestructura.

Esto ya está reflejado en tu propio proyecto: `OrdersController` (capa API) llama a `OrderService` (capa de aplicación/negocio), que llama a `IOrderRepository` (una abstracción) — el servicio de negocio nunca sabe si detrás hay MySQL, SQL Server o una lista en memoria.

**Señal de alerta (violación del principio):** un controller con un `if` de lógica de negocio adentro, o una entidad de dominio con un atributo de Entity Framework específico de la base de datos. Cuando eso aparece, la capa que debería ser independiente dejó de serlo.

## Guard Clauses

En vez de anidar condicionales, se valida temprano y se corta la ejecución apenas algo no es válido.

```csharp
// Sin Guard Clause: anidación innecesaria, difícil de leer
public void ActualizarPedido(Order pedido, int nuevaCantidad)
{
    if (pedido != null)
    {
        if (pedido.Estado != EstadoPedido.Completado)
        {
            if (nuevaCantidad > 0)
            {
                pedido.Cantidad = nuevaCantidad;
            }
        }
    }
}

// Con Guard Clauses: se valida y se corta apenas algo falla, sin anidar
public void ActualizarPedido(Order pedido, int nuevaCantidad)
{
    if (pedido is null)
        throw new ArgumentNullException(nameof(pedido));

    if (pedido.Estado == EstadoPedido.Completado)
        throw new BusinessRuleException("No se puede modificar un pedido completado");

    if (nuevaCantidad <= 0)
        throw new ArgumentException("La cantidad debe ser mayor a 0");

    pedido.Cantidad = nuevaCantidad;
}
```

La segunda versión se lee de arriba hacia abajo sin perder el hilo, y cada condición de error queda explícita en su propia línea — nada de contar niveles de indentación para saber qué se está validando.

## Convenciones de naming en C#

| Elemento | Convención | Ejemplo |
|---|---|---|
| Clases, métodos, propiedades públicas | `PascalCase` | `OrderService`, `GetById()`, `TotalAmount` |
| Variables locales, parámetros | `camelCase` | `pedidoActual`, `nuevaCantidad` |
| Campos privados | `_camelCase` | `_repository`, `_logger` |
| Interfaces | `I` + `PascalCase` | `IOrderRepository` |

No es una regla estética arbitraria — es la convención oficial de Microsoft para C#, y seguirla hace que cualquier desarrollador que entra a un proyecto nuevo reconozca de un vistazo qué es qué, sin tener que leer la declaración completa.

## Cómo cambia en otros lenguajes

PascalCase para clases es prácticamente universal — eso se transfiere igual entre C#, JavaScript, TypeScript, Python y Java. Lo que SÍ cambia, y no es solo estética:

| | C# | Java | Python | JavaScript / TypeScript |
|---|---|---|---|---|
| Métodos / funciones | `PascalCase` | `camelCase` | `snake_case` | `camelCase` |
| Variables / parámetros | `camelCase` | `camelCase` | `snake_case` | `camelCase` |
| Constantes | `PascalCase` | `UPPER_SNAKE_CASE` | `UPPER_SNAKE_CASE` | `UPPER_SNAKE_CASE` (constantes reales) |
| Campos privados | `_camelCase` | `camelCase`, sin prefijo | `_snake_case` (convención, no forzada) | `#campo` (privado real del lenguaje) o `camelCase` sin prefijo |

Si se escribe `calcularTotal` en Python en vez de `calcular_total`, no es un detalle de estilo — un linter de PEP8 lo marca como no conforme, y cualquier desarrollador Python lo nota de inmediato como código "no nativo" del lenguaje.

**El campo privado en Python merece una aclaración:** el guion bajo simple (`_nombre`) es solo una convención — "uso interno", nadie lo impide realmente. El doble guion bajo (`__nombre`) sí dispara un mecanismo real del lenguaje (name mangling: Python lo renombra internamente para evitar colisiones en subclases) — no es idéntico al `private` de C#, pero tampoco es pura decoración.

**En TypeScript/JavaScript moderno**, `#campo` es sintaxis de privacidad real (el motor del lenguaje la hace cumplir, no es solo convención) — a diferencia del `_camelCase` de C#, que es una convención de estilo apoyada sobre un `private` que ya existe como palabra clave separada.

## Comentarios: cuándo sí y cuándo no

- No abusar de los comentarios — no hace falta comentar cada fragmento de código.
- Si el comentario es obvio o repite lo que ya dice el nombre del método/variable, sobra.
- No comentar los cambios realizados (`// se cambió esto el 20/03`) — de eso se encarga Git, no el código.
- No dejar código comentado "por las dudas" — si no se usa, se borra; para recuperarlo está el historial de Git.
- Cuando un comentario sí hace falta (una decisión no obvia, una restricción externa, un workaround puntual), que sea corto y directo.
