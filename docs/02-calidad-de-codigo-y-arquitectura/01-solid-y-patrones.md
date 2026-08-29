# SOLID y Patrones de Diseño

## Los 5 principios SOLID

### S — Single Responsibility Principle

Una clase debe tener una sola razón para cambiar. Si `OrderService` calcula precios, envía emails Y genera facturas, tiene tres razones distintas para cambiar — y tres responsabilidades mezcladas en un solo lugar. En la industria, a este tipo de clase se la conoce como **"Clase Dios" (God Class)** — hace de todo un poco, y por eso cualquier cambio en el sistema termina pasando por ella.

```csharp
// Viola SRP: genera el reporte Y lo guarda en disco — dos responsabilidades, dos razones para cambiar
public class Reporte
{
    public string Generar() => "Contenido del reporte";

    public void GuardarEnArchivo(string contenido) =>
        File.WriteAllText("reporte.txt", contenido);
}

// Respeta SRP: cada clase tiene una única razón para cambiar
public class GeneradorDeReportes
{
    public string Generar() => "Contenido del reporte";
}

public class GuardadorDeArchivos
{
    public void Guardar(string contenido) =>
        File.WriteAllText("reporte.txt", contenido);
}
```

### O — Open/Closed Principle

Un componente debe estar abierto a extensión, cerrado a modificación — agregar comportamiento nuevo no debería requerir tocar código que ya funciona y ya está probado. Lo definió originalmente Bertrand Meyer, y Robert C. Martin lo popularizó dentro de SOLID.

```csharp
// Viola OCP: cada tipo de cliente nuevo obliga a modificar este método
public class CalculadoraDescuentos
{
    public double Calcular(string tipoCliente, double total) => tipoCliente switch
    {
        "Regular" => total * 0.9,
        "Premium" => total * 0.8,
        _ => total
    };
}

// Respeta OCP: un tipo de cliente nuevo se agrega con una clase nueva, sin tocar las existentes
public interface IDescuento
{
    double Aplicar(double total);
}

public class DescuentoRegular : IDescuento
{
    public double Aplicar(double total) => total * 0.9;
}

public class DescuentoPremium : IDescuento
{
    public double Aplicar(double total) => total * 0.8;
}
```

Es, en la práctica, el mismo patrón Strategy que se ve más abajo — OCP es la razón de fondo por la que Strategy existe.

### L — Liskov Substitution Principle

Si `B` hereda de `A`, cualquier lugar que use `A` debe poder recibir un `B` sin que nada se rompa.

```csharp
// Viola LSP: Pinguino hereda de Ave pero no puede cumplir el contrato de "volar"
public class Ave
{
    public virtual void Volar() => Console.WriteLine("Volando");
}

public class Pinguino : Ave
{
    public override void Volar() =>
        throw new NotSupportedException("Los pingüinos no vuelan");
}

void HacerVolar(Ave ave) => ave.Volar(); // explota si recibe un Pinguino

// Respeta LSP: no se fuerza una jerarquía que no tiene sentido en la realidad
public interface IVolador
{
    void Volar();
}

public class Aguila : IVolador
{
    public void Volar() => Console.WriteLine("Volando");
}

public class Pinguino
{
    public void Nadar() => Console.WriteLine("Nadando");
}
```

Señal de alerta clásica de una violación a LSP: un método sobrescrito que lanza `NotImplementedException` o `NotSupportedException` en vez de cumplir lo que promete la clase base.

### I — Interface Segregation Principle

Mejor varias interfaces chicas y específicas que una interfaz gigante que obliga a implementar métodos que no se necesitan — el principio de responsabilidad única, aplicado a interfaces. Una interfaz gigante que mezcla responsabilidades de todo tipo se conoce como **"Fat Interface"**.

```csharp
// Viola ISP: Programador se ve obligado a implementar Disenar() aunque no lo necesita
public interface ITrabajador
{
    void Trabajar();
    void Codificar();
    void Disenar();
}

public class Programador : ITrabajador
{
    public void Trabajar() { }
    public void Codificar() { }
    public void Disenar() => throw new NotImplementedException();
}

// Respeta ISP: interfaces separadas por responsabilidad
public interface ITrabajador
{
    void Trabajar();
}

public interface IDesarrollador
{
    void Codificar();
}

public interface IDisenador
{
    void Disenar();
}

public class Programador : ITrabajador, IDesarrollador
{
    public void Trabajar() { }
    public void Codificar() { }
}
```

### D — Dependency Inversion Principle

Las clases de alto nivel no deben depender de clases concretas de bajo nivel — ambas dependen de una abstracción (interfaz). Por eso `OrderService` depende de `IOrderRepository`, no de `OrderRepository` (la implementación concreta con EF Core).

```csharp
// Viola DIP: Servicio depende directamente de una implementación concreta
public class MySqlRepositorio
{
    public void Guardar(string dato) { /* ... */ }
}

public class Servicio
{
    private readonly MySqlRepositorio _repo = new(); // acoplado a MySQL específicamente

    public void Ejecutar() => _repo.Guardar("dato");
}

// Respeta DIP: Servicio depende de una abstracción, la implementación se inyecta
public interface IRepositorio
{
    void Guardar(string dato);
}

public class MySqlRepositorio : IRepositorio
{
    public void Guardar(string dato) { /* ... */ }
}

public class Servicio
{
    private readonly IRepositorio _repo;

    public Servicio(IRepositorio repo) => _repo = repo; // inyección por constructor

    public void Ejecutar() => _repo.Guardar("dato");
}
```

Existen 3 formas de inyectar una dependencia: **por constructor** (la más común, y la recomendada por defecto), **por propiedad**, y **por parámetro de método**. Ya está aplicado en tu propio proyecto: `OrderService` recibe `IOrderRepository` por constructor, nunca crea un `OrderRepository` concreto adentro.

Esto también conecta con [03-estrategia-de-testing.md](./03-estrategia-de-testing.md): como las pruebas unitarias no deben depender de una base de datos real, DIP es justamente lo que hace posible reemplazar `IOrderRepository` por un mock en los tests — sin esta inversión, sería imposible testear `OrderService` de forma aislada.

## Lo más importante: "justificado", no "porque se puede"

Un patrón de diseño resuelve un problema concreto. Aplicarlo sin ese problema presente es la versión de código de sobre-ingeniería — ver [Simplicidad vs Flexibilidad](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md#2-simplicidad-vs-flexibilidad): código "por si algún día se necesita" que nadie pidió, más difícil de entender, sin beneficio real hoy.

La pregunta antes de aplicar cualquier patrón: **¿qué problema concreto tengo ahora que este patrón resuelve?** Si la respuesta es "ninguno, pero por si acaso", no corresponde todavía.

**Ejemplo concreto, justificado vs. no justificado:**

Supongamos que hoy la API solo necesita mandar notificaciones por email. Dos escenarios posibles:

- **No justificado:** se arma una interfaz `INotificationSender` con implementaciones para Email, SMS, Push y Slack, "por si en el futuro se necesitan". Nadie pidió esos otros 3 canales, y puede que nunca se pidan. Es complejidad que se paga hoy, a cambio de un beneficio que quizás nunca llegue — viola YAGNI (ver [07-principios-de-simplicidad.md](./07-principios-de-simplicidad.md)).
- **Justificado:** el ticket dice explícitamente "esta semana se agrega SMS además de email, y ya está confirmado que el próximo sprint viene push notifications". Ahí SÍ hay un cambio real y confirmado — armar la interfaz ahora evita reescribir todo cuando llegue el segundo canal.

La pregunta no es "¿esto podría cambiar en el futuro?" (casi todo PODRÍA cambiar, eso no dice nada). La pregunta es **"¿hay YA una señal concreta — un ticket, un requerimiento confirmado, un patrón de cambio que ya se repitió — de que va a cambiar?"**. Sin esa señal concreta, es solo una suposición, y no alcanza para justificar la abstracción.

**Aclaración:** una abstracción puede representar fielmente un concepto real del dominio (eso es la Abstracción como pilar de POO) y aun así tener un costo — "representar bien la realidad" y "no tener costo" son cosas distintas. El costo no depende de si la abstracción es válida, depende de tener que saltar de la interfaz a la implementación para leer el comportamiento real.

**La Paradoja de la Flexibilidad:** cada abstracción que se agrega (una interfaz, un patrón de diseño) tiene un costo real de indirección — para entender qué hace el código, hay que saltar de la interfaz a su implementación concreta, en vez de leerlo todo de corrido. Ese salto mental no es gratis, y se paga cada vez que alguien lee ese código.

**Regla de oro para decidir cuándo SÍ introducir una abstracción:** solo cuando se cumple alguna de estas tres condiciones:

1. Ya se siente el dolor del acoplamiento: modificar una sola cosa obliga a tocar muchas partes del código que no deberían verse afectadas.
2. Se sabe con certeza que esa pieza va a cambiar seguido (no "capaz cambia", sino que ya se anticipa el cambio).
3. Hace falta aislar ese componente para poder testearlo (ver [03-estrategia-de-testing.md](./03-estrategia-de-testing.md)) — aplica incluso si esa pieza casi no cambia nunca, como una pasarela de pagos externa.

Si ninguna de las tres aplica todavía, la abstracción es costo sin beneficio — YAGNI en acción (ver [07-principios-de-simplicidad.md](./07-principios-de-simplicidad.md)).

## ¿Cuál patrón, para qué síntoma?

Antes de ver cada uno en detalle, la forma más rápida de elegir es reconocer el síntoma en el propio código, no memorizar los 4 patrones de corrido:

| Si el síntoma es... | El patrón es... |
|---|---|
| Crear un objeto es complejo, o depende de una condición en tiempo de ejecución (`switch`/`if` para decidir qué construir) | **Factory** |
| Hay más de una forma de hacer lo mismo, y esa forma puede cambiar o intercambiarse en tiempo de ejecución | **Strategy** |
| Un cambio de estado necesita avisarle a varios interesados, sin que el objeto que cambia los conozca de antemano | **Observer** |
| La lógica de negocio necesita dejar de depender directamente de EF Core / SQL para acceder a los datos | **Repository** |
| Hace falta desacoplar quién PIDE una acción de quién la EJECUTA, para poder auditarla, encolarla o deshacerla | **Command** |

Si el código no muestra ninguno de estos síntomas todavía, ningún patrón corresponde todavía — ver la sección anterior sobre "justificado".

## Factory

**Problema que resuelve:** la creación de un objeto es compleja o depende de una condición en tiempo de ejecución.

```csharp
public interface INotificationFactory
{
    INotifier Create(NotificationChannel channel);
}

public class NotificationFactory : INotificationFactory
{
    public INotifier Create(NotificationChannel channel) => channel switch
    {
        NotificationChannel.Email => new EmailNotifier(),
        NotificationChannel.Sms => new SmsNotifier(),
        _ => throw new ArgumentException("Canal no soportado")
    };
}
```

**Cuándo NO usarlo:** si el código es simplemente `new Producto()` sin ninguna lógica condicional detrás, un Factory es una capa de indirección que no aporta nada — solo hace más largo el camino para leer el código.

## Strategy

**Problema que resuelve:** hay más de una forma de ejecutar un mismo tipo de operación, y esa forma puede cambiar según el contexto o intercambiarse en tiempo de ejecución.

```csharp
public interface IDiscountStrategy
{
    decimal Calcular(decimal montoOriginal);
}

public class DescuentoClienteVip : IDiscountStrategy
{
    public decimal Calcular(decimal montoOriginal) => montoOriginal * 0.85m;
}

public class SinDescuento : IDiscountStrategy
{
    public decimal Calcular(decimal montoOriginal) => montoOriginal;
}
```

**Cuándo NO usarlo:** si solo existe una forma de calcular algo y no hay ninguna señal de que vaya a cambiar, una interfaz con una sola implementación es complejidad sin beneficio.

## Observer

**Problema que resuelve:** un cambio de estado en un objeto necesita notificar a varios interesados, sin que ese objeto conozca directamente quiénes son.

```csharp
public class Notificador
{
    public event Action? OnCambio;

    public void Cambiar()
    {
        // ... lógica del cambio
        OnCambio?.Invoke(); // notifica a todos los suscriptos, sin saber quiénes son
    }
}

var notificador = new Notificador();
notificador.OnCambio += () => Console.WriteLine("Algo cambió");
notificador.OnCambio += () => _logger.LogInformation("Cambio detectado");

notificador.Cambiar(); // dispara ambas suscripciones
```

En .NET, esto se ve reflejado en eventos y delegates del lenguaje (como en el ejemplo de arriba), o en el patrón más amplio de **Domain Events** (ej: al crear un pedido, se dispara un evento `PedidoCreado` que un servicio de email y un servicio de facturación escuchan de forma independiente, sin que `OrderService` sepa que existen).

**Cuándo NO usarlo:** si hay un solo consumidor fijo y no se prevén más, llamarlo directo es más simple y más fácil de seguir que armar toda la infraestructura de eventos.

## Repository

**Problema que resuelve:** abstraer el acceso a datos detrás de una interfaz, para que la lógica de negocio no dependa directamente de EF Core o de SQL.

```csharp
public interface IOrderRepository
{
    Task<Order?> GetById(int id);
    Task Add(Order order);
}
```

**Matiz importante que pocos Juniors conocen:** con EF Core, el propio `DbContext` **ya es**, internamente, una implementación de los patrones Repository y Unit of Work — `DbSet<T>` funciona como un repositorio, y `SaveChanges()` como una unidad de trabajo. Envolver un `DbContext` en OTRO Repository genérico (uno que solo delega cada método 1 a 1 a EF Core sin agregar lógica propia) a veces es una capa redundante — "un repositorio sobre otro repositorio". Es una discusión real y válida en la comunidad .NET, no un error si se hace así: pero vale la pena saber que el patrón, aplicado sin criterio sobre EF Core, puede no aportar nada nuevo.

El caso donde SÍ vale la pena, incluso sobre EF Core: cuando el repositorio agrega lógica propia (queries complejas reutilizables, mapeo a otro modelo, o para poder cambiar de motor de persistencia sin tocar la capa de negocio) — no cuando es un simple `return _context.Orders.Find(id);` envuelto en una interfaz.

## Command

**Problema que resuelve:** desacoplar quién PIDE una acción de quién la EJECUTA, encapsulando la petición completa como un objeto — en vez de llamar directo a un método con parámetros sueltos.

```csharp
// El Command: una "caja" inmutable con los datos de la petición (un record es ideal para esto)
public record RegistrarEmpleadoCommand(string NombreCompleto, string Departamento, decimal SalarioBase);

// El Handler: la única clase que sabe qué hacer con ese comando
public class RegistrarEmpleadoCommandHandler
{
    public void Handle(RegistrarEmpleadoCommand comando)
    {
        // lógica de negocio real acá
    }
}

// El controller solo arma el comando y lo despacha — no tiene lógica de negocio
var comando = new RegistrarEmpleadoCommand("Jorge Díaz", "Sistemas", 3500m);
new RegistrarEmpleadoCommandHandler().Handle(comando);
```

**Por qué esto es distinto de llamar un método directo:** al ser un objeto, el Command se puede guardar, loguear para auditoría, meter en una cola para procesar después, o revertir — algo imposible con una llamada de método que se ejecuta y se pierde en el momento.

Una extensión clásica: si el Command implementa también un método `Deshacer()` (además de `Ejecutar()`), y un invocador guarda el historial en una pila (`Stack<ICommand>`), se puede implementar un Ctrl+Z real — es el mecanismo de fondo detrás del undo de editores como Word o Photoshop.

**Cuándo NO usarlo:** si la acción es simple y no hace falta historial, cola ni poder revertirla, es una capa de indirección de más — se llama al método directo.

**Conexión con arquitecturas modernas:** Command es la base de **CQRS** (Command Query Responsibility Segregation) — separar las operaciones que escriben (Commands) de las que leen (Queries) — usado en .NET con librerías como MediatR, donde el controller queda reducido a armar el comando y despacharlo, sin saber quién ni cómo lo procesa.

## SOLID y patrones en otros lenguajes (TS, Python, Java)

### SOLID

- **S, O y D generalizan bien** — en TypeScript, Python o Java, "una función/clase hace una sola cosa", "se extiende sin modificar lo existente" y "se depende de abstracciones" siguen aplicando exactamente igual, sean o no lenguajes puramente orientados a objetos.
- **L e I están atadas al modelo de herencia de clases de POO clásica.** Java y TypeScript (con `class`/`interface`) las aplican prácticamente igual que C#. Python es más flexible con el *duck typing* — muchas veces ni hace falta declarar una interfaz formal (aunque existen `Protocol` y `ABC` para eso) — la idea de fondo (no romper el contrato esperado) sigue vigente, pero el mecanismo para hacerla cumplir es más laxo, no impuesto por el compilador como en C#/Java/TS.

### Patrones de diseño

El punto clave para los tres: **TypeScript, Python y Java tienen funciones de primera clase** (se pueden pasar como valores, como los `delegate`/`Func<T>` de C#). Eso cambia cuándo hace falta el patrón completo:

- **Strategy:** en Java "de libro" se usa interfaz + clases (como en C#). En TypeScript y Python, muchas veces alcanza con pasar una función directamente en vez de armar toda la ceremonia de interfaz + clases — el patrón sigue existiendo conceptualmente, pero la implementación se simplifica.
- **Command:** en TypeScript/Python, un Command a veces es literalmente una función o closure capturando su contexto, sin necesidad de una clase con `Ejecutar()`.
- **Observer:** en Java no hay una palabra clave como `event`/`delegate` de C# — se implementa a mano con interfaces, o con librerías (RxJava). En TypeScript, es central a la plataforma (eventos del DOM, `EventEmitter` de Node, Observables de RxJS).
- **Singleton:** en Python, muchas veces ni se implementa como clase con constructor privado — un módulo de Python ya funciona como singleton por cómo cachea Python los imports.

No es que cambie el nombre del patrón — a veces el patrón completo deja de hacer falta porque el lenguaje ya resuelve ese problema con una feature nativa (funciones de primera clase). Verificar esto por lenguaje, no asumirlo, es el mismo hábito que ya se documentó para el resto del roadmap.

## Singleton (patrón clásico) — el que probablemente conviene evitar

**Problema que dice resolver:** garantizar que una clase tenga una única instancia en toda la aplicación, con un punto de acceso global a ella.

```csharp
public class ConfiguracionApp
{
    private static ConfiguracionApp? _instancia;
    private ConfiguracionApp() { } // constructor privado: nadie puede hacer "new" desde afuera

    public static ConfiguracionApp Instancia => _instancia ??= new ConfiguracionApp();

    public string ApiKey { get; set; } = string.Empty;
}

// Uso en cualquier lado del código:
ConfiguracionApp.Instancia.ApiKey = "abc123";
```

**No confundir con el lifetime Singleton de DI** (ver [06-inyeccion-de-dependencias.md](./06-inyeccion-de-dependencias.md)) — logran un resultado parecido (una sola instancia) por caminos completamente distintos:

- **Patrón Singleton clásico:** la clase se administra a sí misma, con un constructor privado y una propiedad estática.
- **Lifetime Singleton de DI:** el contenedor de dependencias decide cuántas instancias crear — la clase no sabe nada de esto, ni tiene código especial para lograrlo.

**Por qué no conviene implementarlo, en detalle:** hay en realidad **dos problemas de fondo**, no una lista de reglas sueltas.

### Problema 1: acoplamiento directo a una clase concreta, no a una abstracción

Cuando una clase escribe `ConfiguracionApp.Instancia` dentro de su lógica, está llamando directamente a ESA clase específica — no a una interfaz, no a algo reemplazable. Es el mismo problema que DIP ya prohíbe con `new SqlServerRepository()` dentro de un servicio, solo que acá queda escondido detrás de una llamada estática que parece inofensiva. Esto se nota apenas se intenta testear:

```csharp
public class OrderService
{
    public void ProcesarPedido(Order pedido)
    {
        var apiKey = ConfiguracionApp.Instancia.ApiKey; // acoplado a la clase concreta
        // ... usa apiKey para llamar a un servicio externo
    }
}

// Test unitario:
[Fact]
public void ProcesarPedido_DeberiaFuncionar()
{
    var service = new OrderService();
    service.ProcesarPedido(pedidoDePrueba);
    // No hay forma de controlar qué ApiKey usa el test.
    // Si ConfiguracionApp.Instancia.ApiKey está vacío o mal configurado
    // en el entorno de test, el test falla por una razón que
    // no tiene nada que ver con la lógica que se está probando.
}
```

Con inyección por constructor (`IConfiguracionApp` inyectado), el test controla exactamente qué configuración usa, sin depender de un estado global que vive fuera del test — ver [03-estrategia-de-testing.md](./03-estrategia-de-testing.md).

### Problema 2: el estado se comparte entre todo lo que corre en la app, al mismo tiempo

Esto es independiente del problema de testing — pasa incluso si nunca se testea nada. Como hay una sola instancia para TODA la aplicación, si esa instancia tiene algo que cambia, cualquier código que la toque está tocando lo mismo que está usando otro request, de otro usuario, en simultáneo:

```csharp
public class ConfiguracionApp
{
    private static ConfiguracionApp? _instancia;
    private ConfiguracionApp() { }
    public static ConfiguracionApp Instancia => _instancia ??= new ConfiguracionApp();

    public int IntentosDeReintento; // acá está el problema: es mutable y compartido
}

// Request del Usuario A, en un hilo:
ConfiguracionApp.Instancia.IntentosDeReintento = 3;

// Al mismo tiempo, Request del Usuario B, en otro hilo, misma instancia del proceso:
ConfiguracionApp.Instancia.IntentosDeReintento = 1;

// Cuando el código del Usuario A vuelve a leer IntentosDeReintento,
// puede encontrarse con el valor que dejó el Usuario B — sin que
// ninguno de los dos haya hecho nada "mal" en su propio código.
```

En una API que atiende requests en paralelo (que es literalmente cómo funciona ASP.NET Core bajo carga), esto no es un caso raro — es el comportamiento normal esperado del sistema.

### La conclusión concreta

No es que "esté mal por principio" — es que el patrón mezcla dos responsabilidades que conviene mantener separadas: **quién decide cuántas instancias hay** (debería vivir en el contenedor de DI, no en la propia clase) y **si esa instancia tiene estado compartido peligroso** (hay que evitarlo activamente, sea Singleton clásico o lifetime Singleton de DI). El patrón clásico junta ambos problemas en un solo lugar, y los hace invisibles hasta que algo falla en producción bajo carga real.

**Por qué igual conviene conocerlo:** es uno de los patrones originales del libro GoF (Gang of Four), y aparece seguido en entrevistas técnicas — entender por qué se lo evita hoy es tan valioso como saber implementarlo. En .NET moderno, casi cualquier caso donde antes se hubiera usado un Singleton clásico se resuelve hoy registrando el servicio con lifetime Singleton en el contenedor de DI: misma garantía de una sola instancia, sin ninguna de las 4 desventajas de arriba.
