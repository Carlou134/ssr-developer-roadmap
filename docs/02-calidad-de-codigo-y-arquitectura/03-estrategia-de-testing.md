# Estrategia de testing

## En una frase

Una estrategia de testing decide **qué probar, con qué tipo de prueba y cuánto**: muchas pruebas unitarias rápidas sobre las reglas de negocio, menos pruebas de integración sobre el cableado real y pocas E2E sobre los flujos críticos, midiendo una cobertura **útil**, no inflada.

-----

## Antes de empezar

Conviene que ya sepas:

* Por qué depender de abstracciones permite aislar código: [SOLID y patrones](01-solid-y-patrones.md).
* Dónde vive la lógica de negocio: [Arquitectura y límites entre capas](02-arquitectura-y-limites.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Bug:** error o defecto que hace que el sistema se comporte de forma distinta a la esperada.
* **Prueba unitaria:** prueba rápida y aislada de una unidad de comportamiento, sin base de datos ni red.
* **Prueba de integración:** prueba que ejercita varias piezas reales juntas, como un servicio con su base de datos.
* **Prueba E2E (*end-to-end*):** prueba del flujo completo, tal como lo usaría un usuario.
* **Pirámide de testing:** muchas pruebas unitarias en la base, menos de integración y pocas E2E en la punta.
* **Mock / doble de prueba:** versión falsa de una dependencia que se usa en una prueba en lugar de la real.
* **Regresión:** algo que funcionaba y dejó de funcionar por un cambio.
* **TDD (*Test-Driven Development*):** escribir la prueba que falla antes del código, luego el código mínimo y después refactorizar.
* **BDD (*Behavior-Driven Development*):** describir el comportamiento esperado en lenguaje de negocio, a menudo con Given/When/Then.
* **Cobertura de código:** porcentaje del código que se ejecutó durante las pruebas.
* **CI (*Continuous Integration*):** ejecutar automáticamente la compilación y las pruebas en cada cambio que se sube al repositorio.

-----

## El problema

Un equipo celebra: el reporte de cobertura marca **94 %**. Una semana después, producción cobra mal los pedidos con descuento.

Revisan las pruebas y encuentran cosas como esta:

```csharp
[Fact]
public void Test1()
{
    var dto = new PedidoDto { Id = 1, Total = 100 };
    Assert.Equal(1, dto.Id);
}

[Fact]
public void ProcesarPedido_Funciona()
{
    var service = new OrderService(new Mock<IOrderRepository>().Object);
    service.Procesar(new Order { Subtotal = 100 }, "Premium");
    // sin Assert
}
```

Y la única prueba "de integración" mockea el repositorio, así que nadie notó que la configuración de Entity Framework guardaba el descuento en una columna con 0 decimales.

Las consecuencias:

* el 94 % medía líneas **ejecutadas**, no reglas **verificadas**: la prueba de `Procesar` pasa por el código del descuento sin comprobar nada;
* se probó a fondo lo que no tiene riesgo (un DTO) y no se probó lo que sí (el cálculo y la persistencia);
* la falsa sensación de seguridad hizo que nadie revisara el cambio con cuidado.

-----

## Cómo funciona

### 1. Pruebas manuales frente a automatizadas

Probar es comparar el comportamiento real con el esperado.

* **Manuales:** una persona usa la aplicación (hace clic, escribe, navega) y compara lo que ve con lo que debería pasar.
* **Automatizadas:** un programa hace esa comparación. Son más rápidas, no se cansan ni se distraen, y se mantienen como cualquier otro código.

Un estudio del NIST (2002) estimó el costo de los bugs de software para la economía de Estados Unidos en unos **59 500 millones de dólares anuales**. La cifra es vieja, pero muestra la escala del problema.

El código de prueba va en un **proyecto separado** (por ejemplo, `OrderManagementBackend.Tests`), con archivos que reflejan qué prueban: `OrderServiceTests.cs` prueba `OrderService.cs`.

### 2. Las pruebas como documentación y red contra regresiones

Una prueba bien escrita hace dos cosas: verifica que algo funciona y **documenta** qué se espera de ese código. Un nombre como `CambiarCantidad_PedidoCompletado_LanzaExcepcion` le dice a cualquiera qué regla existe, sin abrir el código de producción.

Una **regresión** es algo que ya funcionaba y deja de funcionar, normalmente como efecto secundario de un cambio en otra parte. Ejecutar las pruebas después de cada cambio es lo que la detecta: si algo "regresó", al menos una prueba debería fallar antes de llegar a producción.

Por eso refactorizar (y aplicar la Regla del Boy Scout de [Deuda técnica](04-deuda-tecnica.md)) solo es seguro si hay pruebas que avisen cuando algo se rompe.

### 3. TDD y BDD

| | TDD | BDD |
| --- | --- | --- |
| Qué es | Escribir la prueba **antes** del código: rojo (falla), verde (código mínimo), refactor | Describir el comportamiento esperado en lenguaje de negocio, con Given/When/Then |
| Unidad que prueba | Una función o clase: un comportamiento técnico puntual | Una **característica** (*feature*): una funcionalidad completa desde la perspectiva del usuario |
| Quién escribe los casos | El equipo de desarrollo | Desarrollo junto con producto o negocio |

No son excluyentes: BDD define el **qué** debería pasar en lenguaje de negocio, y TDD es la disciplina para construir el código que lo cumple. Hay otras metodologías "pruebas primero" con el mismo espíritu, como **ATDD** (*Acceptance Test-Driven Development*) y **SBE** (*Specification by Example*).

### 4. TDD en la teoría y en la práctica

TDD estricto (prueba antes de cada línea de código, ciclo rojo-verde-refactor sostenido) **no es la práctica dominante en la industria**, aunque los cursos lo presenten como el estándar.

| Lo que se enseña | Lo que se ve en la mayoría de los equipos |
| --- | --- |
| TDD estricto | **Test-after**: se escribe el código y después las pruebas, normalmente en el mismo PR |
| BDD con herramientas dedicadas (Cucumber, Reqnroll, antes SpecFlow) | Pruebas de xUnit con nombres estilo Given/When/Then, sin el tooling completo |
| — | Casi universal: una suite de pruebas automatizadas que corre en CI en cada Pull Request |

**Por qué existe la brecha:** TDD estricto exige una disciplina difícil de sostener bajo presión de fechas, y funciona mejor cuando el problema está bien definido. Es mucho más difícil de aplicar mientras todavía se explora cómo resolver algo.

**Lo que de verdad importa:** que exista una suite sólida que cubra las reglas de negocio reales. Que la prueba se haya escrito antes o después importa mucho menos que el hecho de que exista y proteja algo real.

### 5. Los 3 niveles y la pirámide

| Tipo | ¿Qué prueba? | ¿Cuándo corre? |
| --- | --- | --- |
| **Unitarias** | Reglas de negocio aisladas: calcular un descuento, validar una regla | En local mientras desarrollas y en CI en cada commit o PR. Milisegundos |
| **Integración** | La interacción real entre componentes: un repositorio contra una base de prueba, un endpoint que devuelve el JSON correcto | En CI antes de fusionar a la rama principal. Segundos |
| **E2E** | El flujo completo del usuario, desde la interfaz o la API pública hasta la base de datos (login → carrito → pago) | Para funcionalidades críticas, a menudo en pipelines nocturnos o antes de un release. Minutos. Se automatizan con Playwright o Cypress |

```text
           /\        E2E: pocas, lentas, frágiles, prueban el sistema completo
          /  \
         /----\      Integración: base real, HTTP real; más confianza en el cableado
        /      \
       /--------\    Unitarias: muchas, rápidas, reglas de negocio
```

A medida que se sube, aumenta el costo (tiempo, cómputo, mantenimiento) y el alcance de lo que se prueba, así que conviene tener **menos** pruebas en cada nivel superior. No compiten: las unitarias te dicen rápido **qué regla** se rompió; las de integración, si las piezas **funcionan juntas**.

### 6. Qué reemplazar con dobles en cada nivel

| Nivel | Qué se reemplaza | Qué corre de verdad |
| --- | --- | --- |
| Unitaria | Lo lento o no determinista: base de datos, red, disco, reloj | La unidad de comportamiento bajo prueba |
| Integración | Solo los servicios **externos** que no controlas: una API de terceros, el proveedor de pagos | Todo lo interno: el servicio con su repositorio real, contra una base de prueba |
| E2E | Idealmente nada (a lo sumo, servicios externos de pago o de email en un entorno de prueba) | Todo |

> **Ojo con las dos escuelas de pruebas unitarias:** en la escuela **clásica** (Detroit), la "unidad" es un comportamiento y puede involucrar varias clases reales del dominio; solo se reemplaza lo externo. En la escuela **londinense** (*mockist*), cada clase se prueba sola y **todas** sus colaboradoras se reemplazan por mocks. La clásica produce pruebas más resistentes a refactors; la londinense, fallos más localizados, pero pruebas acopladas a la implementación.

Error común: una prueba de integración que mockea sus propios componentes internos no es de integración, es una unitaria más lenta. Es exactamente lo que pasó en el problema.

### 7. Los 4 criterios de cobertura de código

"Cobertura" no es un solo número. Hay criterios distintos, de menos a más exigente:

| Criterio | Pregunta |
| --- | --- |
| **Funciones** | ¿Se llamó a cada función al menos una vez? |
| **Sentencias** | ¿Se ejecutó cada línea al menos una vez? |
| **Ramas** (*branch*) | ¿Se tomó cada rama de cada decisión (el `if` y el `else`)? |
| **Condiciones** | ¿Cada subexpresión booleana se evaluó como verdadera **y** como falsa? |

```csharp
public int? Sumar(int? x, int? y)
{
    if (x.HasValue && y.HasValue)
        return x + y;
    return null;
}
```

* `Sumar(1, 2)` da 100 % de cobertura de **funciones**, pero deja sin ejecutar `return null`.
* Agregar `Sumar(1, null)` ejecuta las dos ramas: 100 % de **sentencias** y de **ramas**.
* Para 100 % de **condiciones**, `x.HasValue` y `y.HasValue` tienen que ser verdaderas y falsas cada una. Con `Sumar(1, 2)`, `Sumar(1, null)` y `Sumar(null, 1)` alcanza. Un cuarto caso `Sumar(null, null)` no agrega nada: `&&` corta en cortocircuito y `y.HasValue` ni se evalúa.

**¿Con los 4 criterios al 100 % ya no hay bugs?** No. Ninguno de esos casos prueba valores límite como `Sumar(int.MaxValue, 1)`, que en C# **se desborda en silencio** y devuelve un número negativo, salvo que el código esté en un bloque `checked`. La cobertura dice que el código **se ejecutó**, no que se probó con los valores correctos.

> **Sintaxis frente a comportamiento:** la versión original de este ejemplo, en JavaScript, muestra que `sumar('1', '2')` devuelve `'12'` por la conversión implícita de tipos. En C#, el tipado estático evita **ese** bug, porque no compila. Pero no significa que C# no tenga sorpresas: el desborde silencioso es el equivalente que sí aplica aquí.

### 8. Cobertura de código frente a cobertura de pruebas

| | Cobertura de código | Cobertura de pruebas |
| --- | --- | --- |
| Mide | Porcentaje de **código** ejecutado durante las pruebas | Porcentaje de **requerimientos** cubiertos por al menos una prueba |
| Puede ser alta y aun así... | No verificar ningún resultado | Probar cada requerimiento con un solo caso feliz |

Un sistema puede tener 100 % de cobertura de código y baja cobertura de pruebas, si nunca se validó un requerimiento real. Ninguna de las dos garantiza ausencia de bugs: dan confianza, no certeza.

### 9. Cobertura útil, no inflada

Es fácil inflar el número probando lo que no tiene riesgo: getters y setters automáticos, DTOs sin lógica. Sube el porcentaje sin agregar protección.

| Prioriza | Prioriza menos (o no pruebes) |
| --- | --- |
| Reglas de negocio: si se rompen, hay un problema real | Getters y setters automáticos |
| Casos borde: valores límite, vacíos, nulos | Código generado automáticamente |
| Código que cambia seguido | Envoltorios que solo delegan a una librería |

Peor que la cobertura inflada es el **coverage theater**: pruebas que ejecutan líneas sin verificar nada, solo para que la herramienta las marque como cubiertas. Dan una falsa sensación de seguridad, y eso es peor que no tener la prueba.

### 10. ¿Cuánta cobertura hace falta?

Una referencia citada en la industria, de la guía de buenas prácticas de cobertura de Google (publicada en su blog de testing en 2020):

| Cobertura | Valoración |
| --- | --- |
| 60 % | Aceptable |
| 75 % | Recomendable |
| 90 % | Ejemplar |

No es una ley, pero es mucho más realista que perseguir el 100 %. La excepción es el software regulado o crítico (aviónica, dispositivos médicos, el núcleo de sistemas financieros), donde las certificaciones exigen coberturas muy altas.

En un backend real, la cobertura es **despareja, y está bien**:

* **Alta (90-100 %):** dominio y aplicación, es decir, reglas de negocio, validadores y servicios. Es barata de probar y es donde más importa.
* **Baja o nula:** DTOs, mapeos triviales, `Program.cs`, controllers que solo delegan.

Un 68 % total en un proyecto sano es normal: es el promedio de las dos cosas.

Recomendación práctica:

1. Pon el foco en dominio y aplicación: ahí apunta a 80-90 %.
2. Usa el número como **piso en CI, no como objetivo a maximizar**: "no se fusiona si la cobertura de esta capa baja del 70 %", no "hay que llegar al 95 % como sea".
3. Mide por proyecto o namespace (SonarQube lo permite, ver [Deuda técnica](04-deuda-tecnica.md)): exige 80 % en dominio y aplicación y nada en infraestructura y API.

-----

## Ejemplo completo

El mismo servicio del problema, con una prueba por nivel donde aporta.

`Order.cs` (la regla a proteger):

```csharp
public class Order
{
    public decimal Subtotal { get; init; }
    public decimal Total { get; private set; }

    public void AplicarDescuento(decimal porcentaje)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(porcentaje);
        ArgumentOutOfRangeException.ThrowIfGreaterThan(porcentaje, 1m);

        Total = Math.Round(Subtotal * (1 - porcentaje), 2);
    }
}
```

**Unitarias** (`tests/Tienda.UnitTests/OrderTests.cs`): la regla de negocio, con sus límites.

```csharp
public class OrderTests
{
    [Theory]
    [InlineData(100, 0.2, 80)]
    [InlineData(99.99, 0.15, 84.99)]
    [InlineData(100, 0, 100)]
    [InlineData(100, 1, 0)]
    public void AplicarDescuento_PorcentajeValido_CalculaElTotal(decimal subtotal, decimal porcentaje, decimal esperado)
    {
        var pedido = new Order { Subtotal = subtotal };

        pedido.AplicarDescuento(porcentaje);

        Assert.Equal(esperado, pedido.Total);
    }

    [Theory]
    [InlineData(-0.1)]
    [InlineData(1.1)]
    public void AplicarDescuento_PorcentajeFueraDeRango_LanzaExcepcion(decimal porcentaje)
    {
        var pedido = new Order { Subtotal = 100 };

        Assert.Throws<ArgumentOutOfRangeException>(() => pedido.AplicarDescuento(porcentaje));
    }
}
```

**Integración** (`tests/Tienda.IntegrationTests/OrderRepositoryTests.cs`): la persistencia real, contra un SQL Server en un contenedor con Testcontainers.

```csharp
public class OrderRepositoryTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sql = new MsSqlBuilder().Build();

    public Task InitializeAsync() => _sql.StartAsync();
    public Task DisposeAsync() => _sql.DisposeAsync().AsTask();

    [Fact]
    public async Task Guardar_PedidoConDecimales_ConservaElTotal()
    {
        var opciones = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_sql.GetConnectionString())
            .Options;

        await using (var db = new AppDbContext(opciones))
        {
            await db.Database.EnsureCreatedAsync();
            var pedido = new Order { Subtotal = 99.99m };
            pedido.AplicarDescuento(0.15m);
            db.Orders.Add(pedido);
            await db.SaveChangesAsync();
        }

        await using (var db = new AppDbContext(opciones))
        {
            var guardado = await db.Orders.SingleAsync();
            Assert.Equal(84.99m, guardado.Total);
        }
    }
}
```

Esta es la prueba que habría detectado la columna sin decimales del problema: con un mock, el bug era invisible.

**E2E:** una sola prueba con Playwright del flujo "agregar al carrito → aplicar cupón → pagar", ejecutada en el pipeline nocturno.

Ejecución:

```text
dotnet test tests/Tienda.UnitTests
dotnet test tests/Tienda.IntegrationTests
```

Las unitarias tardan milisegundos y corren en cada commit; las de integración tardan unos segundos (más el arranque del contenedor) y corren en cada PR.

Qué observar:

* Las pruebas unitarias se concentran en los **límites** (0, 1, valores con decimales), no en el "caso del medio".
* La prueba de integración usa un SQL Server real: es la única que puede detectar errores de mapeo o de tipos de columna.
* No hay ninguna prueba para `PedidoDto`: no tiene lógica.
* Cada `Assert` verifica la regla que el nombre de la prueba promete.

-----

## Errores comunes

**1. Perseguir el porcentaje de cobertura.**
Qué pasa: pruebas sobre DTOs y getters, y reglas de negocio sin verificar.
Por qué: se optimiza la métrica en lugar del objetivo.
Arreglo: pisos de cobertura por capa y revisión de qué verifica cada prueba en el code review.

**2. Pruebas sin `Assert` (coverage theater).**
Qué pasa: todo verde aunque el código esté mal.
Por qué: la prueba ejecuta el código pero no comprueba ningún resultado.
Arreglo: cada prueba verifica un resultado o efecto. Rompe el código a propósito y comprueba que la prueba falla.

**3. Pruebas de integración que mockean lo interno.**
Qué pasa: los errores de configuración, mapeo o consultas llegan a producción.
Por qué: se reemplaza el repositorio o el `DbContext` "para que sea más rápido".
Arreglo: en integración, solo se reemplaza lo externo; la base de datos es real (Testcontainers o una instancia de prueba).

**4. Usar EF Core InMemory como si fuera una base de datos real.**
Qué pasa: las pruebas pasan y en SQL Server falla una consulta, una restricción o una transacción.
Por qué: el proveedor InMemory no es relacional: no aplica restricciones, no traduce a SQL y no se comporta como el motor real.
Arreglo: para pruebas de integración, usa el mismo motor que producción (por ejemplo, con Testcontainers).

**5. Demasiadas pruebas E2E.**
Qué pasa: un pipeline de 40 minutos con pruebas que fallan "a veces".
Por qué: se prueban por la interfaz reglas que se podían verificar con una prueba unitaria.
Arreglo: baja cada verificación al nivel más bajo que la pueda hacer; deja las E2E para los flujos críticos.

-----

## Evolución y versiones

* **2002:** el NIST estima el costo de los bugs en 59 500 millones de dólares anuales en EE. UU.
* **2003:** Kent Beck publica *Test-Driven Development: By Example*.
* **2006:** Dan North presenta BDD y el formato Given/When/Then.
* **2009:** Mike Cohn populariza la pirámide de testing en *Succeeding with Agile*.
* **2020:** Google publica sus buenas prácticas de cobertura (60 / 75 / 90 %).
* **2024:** SpecFlow llega al fin de su soporte; la comunidad continúa el proyecto como **Reqnroll**.

-----

## Cuándo sí y cuándo no

**Escribe pruebas unitarias cuando:** hay reglas de negocio, cálculos, validaciones o decisiones con varios caminos.

**Escribe pruebas de integración cuando:** el riesgo está en el cableado: consultas, mapeos del ORM, serialización, configuración.

**Escribe pruebas E2E cuando:** el flujo es crítico para el negocio (pagar, registrarse) y ninguna prueba de nivel inferior da suficiente confianza.

**No escribas pruebas cuando:** el código no tiene lógica (DTOs, propiedades automáticas) o es generado. Ahí no protegen nada.

-----

## Resumen en 5 líneas

1. Pirámide: muchas unitarias, menos de integración, pocas E2E.
2. En unitarias se reemplaza lo externo y lento; en integración, solo lo externo que no controlas.
3. La cobertura mide código ejecutado, no reglas verificadas: úsala como piso por capa, no como objetivo.
4. Prioriza reglas de negocio, casos borde y código que cambia seguido.
5. TDD estricto es poco común en la práctica; lo indispensable es una suite sólida que corra en CI.

-----

## Para profundizar

<details>
<summary>Mutation testing: ¿mis pruebas detectan errores?</summary>

La cobertura dice qué líneas se ejecutaron. El **mutation testing** responde otra pregunta: si el código tuviera un error, ¿alguna prueba fallaría? Una herramienta como **Stryker.NET** introduce cambios pequeños en el código (cambia `>=` por `>`, `+` por `-`) y ejecuta las pruebas. Si ninguna falla, el "mutante sobrevivió": hay código ejecutado pero no verificado.

```text
dotnet tool install -g dotnet-stryker
dotnet stryker
```

Es la mejor defensa contra el coverage theater, aunque es lenta: conviene aplicarla al dominio, no a todo el proyecto.

</details>

<details>
<summary>Probar una API completa con <code>WebApplicationFactory</code></summary>

Para pruebas de integración de una API ASP.NET Core, `WebApplicationFactory<Program>` (paquete `Microsoft.AspNetCore.Mvc.Testing`) levanta la aplicación en memoria con toda su configuración real y te da un `HttpClient`:

```csharp
public class PedidosApiTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task GetPedido_Inexistente_Devuelve404()
    {
        var client = factory.CreateClient();

        var respuesta = await client.GetAsync("/api/pedidos/999");

        Assert.Equal(HttpStatusCode.NotFound, respuesta.StatusCode);
    }
}
```

Con `WithWebHostBuilder` puedes reemplazar solo los servicios externos (la pasarela de pagos) y dejar el resto real.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Hay tres niveles: unitarias, que prueban una pieza aislada; de integración, que prueban varias piezas juntas, como un servicio con la base de datos; y E2E, que prueban el flujo completo. Se usan muchas unitarias y pocas E2E, como en una pirámide.

### Respuesta ampliada (semi-senior)

Sigo la pirámide: muchas unitarias sobre las reglas de negocio, aisladas de lo externo; pruebas de integración contra una base real, por ejemplo con Testcontainers, porque ahí aparecen los errores de mapeo y de consultas que un mock oculta; y pocas E2E para los flujos críticos. No persigo un número de cobertura global: pongo pisos por capa en CI, alto en dominio y aplicación, y nada en DTOs. Sé que la cobertura mide código ejecutado y no reglas verificadas, así que reviso que cada prueba tenga un `Assert` significativo. En la práctica escribo pruebas junto con el código, y uso TDD cuando las reglas están claras.

### Preguntas frecuentes de seguimiento

**1. ¿100 % de cobertura significa que no hay bugs?**
No. Significa que cada línea se ejecutó, no que se verificó con los valores correctos. Puede haber pruebas sin `Assert` o casos límite sin probar.

**2. ¿Qué mockearías en una prueba de integración?**
Solo los servicios externos que no controlo, como una API de pagos. La base de datos y los componentes internos tienen que ser reales; si no, es una prueba unitaria lenta.

**3. ¿Por qué no usar el proveedor InMemory de EF Core?**
Porque no es relacional: no aplica restricciones ni traduce a SQL, así que una consulta puede pasar en la prueba y fallar en SQL Server.

**4. ¿Haces TDD?**
Respuesta honesta: lo uso cuando las reglas están claras, porque me obliga a pensar la API desde quien la usa. Cuando estoy explorando, escribo las pruebas después, pero en el mismo PR. Lo que no negocio es que las reglas de negocio queden cubiertas.

-----

## Práctica

**Ejercicio 1.** ¿Qué casos de prueba mínimos necesitas para 100 % de cobertura de condiciones en este método?

```csharp
public bool PuedeComprar(bool mayorDeEdad, bool tieneSaldo) => mayorDeEdad && tieneSaldo;
```

<details>
<summary>Solución</summary>

Cada condición tiene que evaluarse como verdadera y como falsa:

* `(true, true)`: `mayorDeEdad` = V, `tieneSaldo` = V.
* `(true, false)`: `tieneSaldo` = F.
* `(false, true)`: `mayorDeEdad` = F (por cortocircuito, `tieneSaldo` ni se evalúa).

Tres casos alcanzan. `(false, false)` no agrega cobertura de condiciones, aunque puedes incluirlo si documenta una regla de negocio relevante.

</details>

**Ejercicio 2.** ¿Qué tiene de malo esta prueba "de integración"? Reescríbela.

```csharp
[Fact]
public async Task CrearPedido_Integracion()
{
    var repo = new Mock<IOrderRepository>();
    var pagos = new Mock<IPasarelaPagos>();
    var service = new OrderService(repo.Object, pagos.Object);

    await service.CrearAsync(new CrearPedidoRequest(ProductoId: 1, Cantidad: 2));

    repo.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Once);
}
```

<details>
<summary>Solución</summary>

* Mockea el repositorio, que es un componente **interno**: no prueba ninguna integración real. Es una prueba unitaria con un nombre engañoso.
* Solo verifica que se llamó a `AddAsync`, no que el pedido se guardó con los datos correctos.

Versión de integración: repositorio y base de datos reales, y solo la pasarela de pagos (externa) reemplazada.

```csharp
[Fact]
public async Task CrearAsync_PedidoValido_QuedaGuardadoConSuTotal()
{
    await using var db = new AppDbContext(_opcionesConContenedorSql);
    await db.Database.EnsureCreatedAsync();
    var pagos = new Mock<IPasarelaPagos>();
    var service = new OrderService(new EfOrderRepository(db), pagos.Object);

    var id = await service.CrearAsync(new CrearPedidoRequest(ProductoId: 1, Cantidad: 2));

    var guardado = await db.Orders.SingleAsync(o => o.Id == id);
    Assert.Equal(2, guardado.Cantidad);
}
```

</details>

-----

## Siguiente lección

[Deuda técnica](04-deuda-tecnica.md)
