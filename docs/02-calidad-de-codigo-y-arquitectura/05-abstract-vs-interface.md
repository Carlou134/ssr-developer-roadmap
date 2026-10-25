# Clase abstracta frente a interfaz

## En una frase

Una clase abstracta es **una base compartida para tipos que "son un"** algo y comparten estado y lógica; una interfaz es **un contrato de capacidades que "puede hacer"** cualquier tipo, sin importar su jerarquía. Alrededor de esa decisión aparecen otras tres: `sealed`, sobrecarga y sobrescritura.

-----

## Antes de empezar

Conviene que ya sepas:

* Herencia y el principio de sustitución de Liskov: [SOLID y patrones](01-solid-y-patrones.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Clase abstracta:** clase que no se puede instanciar y sirve de base para otras; puede tener estado, constructores y métodos implementados.
* **Interfaz:** contrato que define qué miembros debe tener un tipo, sin estado de instancia.
* **Default interface method:** método de una interfaz con una implementación por defecto (C# 8).
* **`sealed`:** modificador que impide heredar de una clase o volver a sobrescribir un método.
* **Sobrecarga (*overloading*):** varios métodos con el mismo nombre y distintos parámetros.
* **Sobrescritura (*override*):** una clase hija redefine un método `virtual` o `abstract` de la base.
* **Firma:** nombre de un método más el tipo, la cantidad y el orden de sus parámetros.
* **Despacho dinámico:** decidir en tiempo de ejecución qué implementación de un método se llama, según el tipo real del objeto.

-----

## El problema

Un sistema de pagos empieza así:

```csharp
public abstract class MedioDePago
{
    public abstract void Cobrar(decimal monto);
    public abstract void Reembolsar(decimal monto);
    public abstract void GuardarTarjeta(string numero);
}

public class Tarjeta : MedioDePago { /* implementa las tres */ }

public class Efectivo : MedioDePago
{
    public override void Cobrar(decimal monto) { /* ... */ }
    public override void Reembolsar(decimal monto) { /* ... */ }
    public override void GuardarTarjeta(string numero) =>
        throw new NotSupportedException(); // el efectivo no tiene tarjeta
}
```

Después llega `CuponDeDescuento`, que también "se puede cobrar", pero ya hereda de `Promocion`. C# no permite heredar de dos clases, así que alguien copia la lógica de `Cobrar` en `CuponDeDescuento`.

Las consecuencias:

* `Efectivo` está obligado a implementar algo que no puede hacer (viola LSP e ISP);
* la herencia simple bloqueó a `CuponDeDescuento`, y la lógica se duplicó;
* la jerarquía mezcla lo que los tipos **son** con lo que **pueden hacer**.

-----

## Cómo funciona

### 1. Clase abstracta: "es un"

Una clase base que **no se puede instanciar** directamente, pensada como plantilla para clases que **son del mismo tipo**: comparten propiedades y lógica común.

```csharp
public abstract class Animal
{
    protected Animal(string nombre) => Nombre = nombre;

    public string Nombre { get; }                       // estado compartido

    public void Presentarse() =>                         // lógica compartida
        Console.WriteLine($"Soy {Nombre}");

    public abstract void HacerSonido();                  // obliga a las hijas a implementarlo
}

public class Perro(string nombre) : Animal(nombre)
{
    public override void HacerSonido() => Console.WriteLine("Guau");
}
```

Si lo usas mal:

```csharp
var animal = new Animal("Firulais");
```

```text
error CS0144: Cannot create an instance of the abstract type or interface 'Animal'
```

### 2. Interfaz: "puede hacer"

Un **contrato de capacidades**: define qué miembros debe tener un tipo, sin estado de instancia. Sirve para tipos que no tienen nada en común como jerarquía, pero comparten una capacidad.

```csharp
public interface ICaminante
{
    void Caminar();
}

public class Robot : ICaminante
{
    public void Caminar() => Console.WriteLine("El robot camina");
}

public class Persona : ICaminante
{
    public void Caminar() => Console.WriteLine("La persona camina");
}
```

Un `Robot` y una `Persona` no comparten tipo: no tiene sentido que hereden de una base común. Pero los dos **pueden** caminar, y ahí encaja una interfaz.

> **Ojo, esto cambió en C# 8:** la idea de que una interfaz "no tiene ninguna implementación" ya no es cierta. Desde C# 8 (2019) una interfaz puede tener **métodos con implementación por defecto** y miembros estáticos. Lo que sigue sin poder tener son **campos de instancia** (estado).
>
> ```csharp
> public interface INotificador
> {
>     void Enviar(string mensaje);
>     void EnviarUrgente(string mensaje) => Enviar($"[URGENTE] {mensaje}"); // default
> }
> ```
>
> Se pensó para poder agregar miembros a una interfaz pública sin romper a quienes ya la implementan, no para reemplazar a las clases abstractas.

### 3. Cómo elegir

| | Clase abstracta | Interfaz |
| --- | --- | --- |
| Relación | **Es un** (`Perro` es un `Animal`) | **Puede hacer** (`Robot` puede caminar) |
| Cuántas puede usar un tipo | **Una sola** (herencia simple) | **Varias** |
| Estado (campos de instancia) | Sí | No |
| Constructores | Sí | No |
| Implementación | Sí, y es lo normal | Solo métodos por defecto (C# 8+), uso excepcional |
| Modificadores de acceso en miembros | `protected`, `private`, etc. | Públicos por defecto |
| Ideal para | Compartir estado y lógica entre tipos muy cercanos | Contratos entre capas, dependencias inyectables, capacidades transversales |

Regla práctica: **empieza por una interfaz**. Pasa a una clase abstracta solo cuando varios tipos que realmente "son lo mismo" necesitan compartir estado o lógica. La interfaz no te ata a una jerarquía y es lo que se inyecta y se reemplaza en las pruebas.

Y no son excluyentes: es común una interfaz para el contrato y una clase abstracta que la implementa parcialmente para reutilizar lógica.

```csharp
public interface IExportador { byte[] Exportar(Reporte reporte); }

public abstract class ExportadorBase : IExportador
{
    public byte[] Exportar(Reporte reporte)
    {
        ArgumentNullException.ThrowIfNull(reporte);
        return Serializar(reporte);
    }

    protected abstract byte[] Serializar(Reporte reporte);
}
```

### 4. `sealed`

`sealed` impide que alguien herede de una clase:

```csharp
public sealed class PagoService
{
    // nadie puede hacer "class Otro : PagoService"
}
```

| Modificador | Intención |
| --- | --- |
| `abstract` | "Esto **debe** extenderse": no se puede usar sin una hija |
| `sealed` | "Esto **no** se extiende por herencia" |

Cuándo usarlo: cuando la clase está completa y no hay un caso real para heredarla, o para evitar que se sobrescriba comportamiento crítico (por ejemplo, de seguridad). También tiene una ventaja de rendimiento: el runtime puede evitar el despacho dinámico en llamadas a una clase sellada. Desde .NET 7, el analizador **CA1852** sugiere sellar los tipos `internal` que nadie hereda.

> **Ojo con OCP:** es tentador decir que `sealed` "es OCP" porque la clase queda "cerrada". Pero OCP pide estar cerrado a **modificación** y **abierto** a extensión, y `sealed` cierra justamente la extensión por herencia. Son compatibles cuando la extensión se hace por **composición o interfaces** (por ejemplo, con Strategy) en lugar de por herencia, que es lo que se recomienda hoy.

`sealed` también se usa en un `override` para que las hijas siguientes no puedan volver a sobrescribirlo: `public sealed override void Procesar()`.

### 5. Sobrecarga frente a sobrescritura

Se confunden porque suenan parecido, pero son mecanismos distintos.

**Sobrecarga:** mismo nombre y **distinta firma** (otros parámetros), en la misma clase. El compilador decide cuál llamar según los argumentos.

```csharp
public int Sumar(int a, int b) => a + b;
public double Sumar(double a, double b) => a + b;
public int Sumar(int a, int b, int c) => a + b + c;
```

**Sobrescritura:** una clase hija redefine un método `virtual` o `abstract` de la base, con la **misma firma**. El runtime decide cuál ejecutar según el tipo real del objeto.

```csharp
public class Animal
{
    public virtual void Sonido() => Console.WriteLine("Sonido genérico");
}

public class Perro : Animal
{
    public override void Sonido() => Console.WriteLine("Guau");
}

Animal a = new Perro();
a.Sonido(); // "Guau": se decide en tiempo de ejecución
```

| | Sobrecarga | Sobrescritura |
| --- | --- | --- |
| Dónde | Misma clase | Clase hija |
| Firma | Distinta | Igual |
| Quién decide | El compilador (tiempo de compilación) | El runtime (tiempo de ejecución, despacho dinámico) |
| Palabra clave | Ninguna | `virtual`/`abstract` en la base, `override` en la hija |

Si lo usas mal:

```csharp
public int Calcular(int x) => x;
public double Calcular(int x) => x; // solo cambia el tipo de retorno
```

```text
error CS0111: Type 'Calculadora' already defines a member called 'Calcular' with the same parameter types
```

El tipo de retorno no forma parte de la firma: dos métodos que solo difieren en él no son una sobrecarga válida.

> **Ojo con `new` en lugar de `override`:** si en la hija escribes `public new void Sonido()`, no sobrescribes: **ocultas** el método de la base. Con `Animal a = new Perro(); a.Sonido();` se ejecuta el de `Animal`, porque `new` se resuelve por el tipo de la **variable**, no del objeto. Si olvidas `override` y `new`, el compilador avisa con **CS0114**.

-----

## Ejemplo completo

El sistema de pagos del problema, separando lo que los tipos **son** de lo que **pueden hacer**.

```csharp
// Capacidades: interfaces chicas (ISP)
public interface ICobrable
{
    void Cobrar(decimal monto);
}

public interface IReembolsable
{
    void Reembolsar(decimal monto);
}

public interface IGuardaTarjeta
{
    void GuardarTarjeta(string numero);
}

// Base compartida solo para los medios que realmente "son" medios de pago
public abstract class MedioDePago : ICobrable
{
    private readonly List<decimal> _cobros = [];

    public IReadOnlyList<decimal> Cobros => _cobros;

    public void Cobrar(decimal monto)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(monto);
        EjecutarCobro(monto);
        _cobros.Add(monto); // lógica y estado compartidos
    }

    protected abstract void EjecutarCobro(decimal monto);
}

public sealed class Tarjeta : MedioDePago, IReembolsable, IGuardaTarjeta
{
    protected override void EjecutarCobro(decimal monto) { /* llamar a la pasarela */ }
    public void Reembolsar(decimal monto) { /* ... */ }
    public void GuardarTarjeta(string numero) { /* ... */ }
}

public sealed class Efectivo : MedioDePago, IReembolsable
{
    protected override void EjecutarCobro(decimal monto) { /* registrar en caja */ }
    public void Reembolsar(decimal monto) { /* ... */ }
}

// Ya tiene otra base, pero puede implementar la capacidad
public sealed class CuponDeDescuento : Promocion, ICobrable
{
    public void Cobrar(decimal monto) { /* descontar del saldo del cupón */ }
}
```

Uso:

```csharp
void CobrarTodo(IEnumerable<ICobrable> cobrables, decimal monto)
{
    foreach (var c in cobrables) c.Cobrar(monto);
}

CobrarTodo([new Tarjeta(), new Efectivo(), new CuponDeDescuento()], 100m);
```

Qué observar:

* `Efectivo` ya no implementa `GuardarTarjeta`: no hay ningún `NotSupportedException`.
* `CuponDeDescuento` puede cobrarse aunque herede de `Promocion`, porque `ICobrable` es una interfaz.
* La clase abstracta existe solo para compartir lo que es común de verdad: la validación y el registro de cobros.
* Las hojas de la jerarquía son `sealed`: no hay ningún caso para heredarlas.

-----

## Errores comunes

**1. Clase abstracta para todo.**
Qué pasa: jerarquías profundas y tipos que no pueden heredar de dos bases a la vez.
Por qué: se elige la clase abstracta porque "puede tener código", aunque no haya estado compartido.
Arreglo: empieza por una interfaz y pasa a una clase abstracta solo si hay estado o lógica común real.

**2. Interfaces gordas.**
Qué pasa: implementaciones que lanzan `NotSupportedException`.
Por qué: una interfaz junta capacidades que no todos los tipos tienen.
Arreglo: divide en interfaces chicas por capacidad (ISP).

**3. Usar default interface methods como si fueran una clase base.**
Qué pasa: lógica de negocio repartida entre interfaces, difícil de encontrar y de probar.
Por qué: se usan para "compartir código" en lugar de para evolucionar una interfaz pública.
Arreglo: para compartir lógica, usa una clase abstracta o composición.

**4. Ocultar con `new` creyendo que se sobrescribe.**
Qué pasa: el método de la hija no se ejecuta cuando la variable es del tipo base.
Por qué: `new` se resuelve por el tipo de la variable; `override`, por el tipo real del objeto.
Arreglo: marca el método base como `virtual` y usa `override`; trata la advertencia CS0114 como error.

**5. Sobrecargas ambiguas.**
Qué pasa: `error CS0121: The call is ambiguous between the following methods`.
Por qué: dos sobrecargas aceptan los mismos argumentos con conversiones de igual costo, por ejemplo `Metodo(int, double)` y `Metodo(double, int)` llamados con `(1, 1)`.
Arreglo: diseña sobrecargas con parámetros claramente distintos, o usa nombres distintos.

-----

## Evolución y versiones

* **C# 1.0 (2002):** interfaces solo con firmas; clases abstractas, `sealed`, `virtual` y `override`.
* **C# 8.0 (2019):** *default interface methods*: las interfaces pueden tener implementación y miembros estáticos, pero no estado de instancia.
* **C# 11 (2022):** miembros `static abstract` en interfaces, que habilitan la matemática genérica (`INumber<T>`).
* **.NET 7 (2022):** el analizador CA1852 sugiere sellar los tipos `internal` que no se heredan.

-----

## Cuándo sí y cuándo no

**Usa una interfaz cuando:** defines un contrato entre capas, una dependencia que se inyecta o una capacidad que tienen tipos sin relación entre sí.

**Usa una clase abstracta cuando:** varios tipos que realmente "son lo mismo" comparten estado o lógica, y quieres obligar a cada uno a completar una parte.

**Usa `sealed` cuando:** la clase no está pensada para heredarse. Es un buen valor por defecto para clases concretas.

**No uses herencia cuando:** el único motivo es reutilizar código. Prefiere la composición.

-----

## Resumen en 5 líneas

1. Clase abstracta: "es un", una sola por tipo, puede tener estado y constructores.
2. Interfaz: "puede hacer", varias por tipo, sin estado de instancia (desde C# 8, con métodos por defecto).
3. Empieza por una interfaz; pasa a una clase abstracta solo si hay estado o lógica común real.
4. `sealed` bloquea la herencia; es compatible con OCP si la extensión se hace por composición.
5. Sobrecarga: distinta firma, la resuelve el compilador. Sobrescritura: misma firma, la resuelve el runtime.

-----

## Para profundizar

<details>
<summary>Interfaces en otros lenguajes</summary>

* **Java:** igual que C# desde Java 8, con métodos `default`. Una clase hereda de una sola clase e implementa varias interfaces.
* **TypeScript:** las interfaces son **estructurales**: un objeto cumple la interfaz si tiene la forma correcta, aunque no lo declare con `implements`. Además, desaparecen al compilar a JavaScript: no existen en tiempo de ejecución.
* **Python:** no tiene interfaces como palabra clave. Usa `ABC` (clases base abstractas, que sí se verifican al instanciar) o `Protocol` (tipado estructural que verifica una herramienta como mypy, no el intérprete).
* **Go:** las interfaces se implementan de forma implícita, sin declararlo, como en TypeScript.

El concepto "contrato de capacidades" se transfiere; el **mecanismo** (nominal frente a estructural, en compilación o en ejecución) cambia, y eso es comportamiento, no solo sintaxis.

</details>

<details>
<summary>Implementación explícita de interfaces</summary>

Si una clase implementa dos interfaces con un miembro del mismo nombre, puede implementarlos por separado:

```csharp
public class Archivo : ILector, IEscritor
{
    void ILector.Abrir() { /* modo lectura */ }
    void IEscritor.Abrir() { /* modo escritura */ }
}

((ILector)new Archivo()).Abrir();
```

Los miembros implementados explícitamente solo son accesibles a través de la interfaz. También se usa para "esconder" miembros que no forman parte del uso normal de la clase.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Una clase abstracta es una clase base que no se puede instanciar y puede tener código y propiedades compartidas. Una interfaz es un contrato que define qué métodos debe tener una clase. Una clase puede heredar de una sola clase abstracta, pero implementar varias interfaces.

### Respuesta ampliada (semi-senior)

Uso la clase abstracta para la relación "es un", cuando varios tipos cercanos comparten estado y lógica, y la interfaz para "puede hacer", cuando quiero un contrato que cumplan tipos sin relación entre sí. Por defecto empiezo por interfaces, porque no atan a una jerarquía, permiten implementar varias y son lo que inyecto y reemplazo en pruebas. Sé que desde C# 8 las interfaces pueden tener métodos por defecto, pero los uso para evolucionar una API pública sin romperla, no para compartir lógica. Sello las clases concretas que no están pensadas para heredarse, y prefiero extender por composición antes que por herencia.

### Preguntas frecuentes de seguimiento

**1. ¿Una interfaz puede tener implementación en C#?**
Sí, desde C# 8, con *default interface methods*. Lo que no puede tener es estado de instancia.

**2. ¿Qué diferencia hay entre sobrecarga y sobrescritura?**
La sobrecarga son varios métodos con el mismo nombre y distinta firma, y la resuelve el compilador. La sobrescritura redefine en una clase hija un método `virtual` de la base con la misma firma, y la resuelve el runtime según el tipo real del objeto.

**3. ¿Para qué sirve `sealed`?**
Para impedir la herencia de una clase o que se vuelva a sobrescribir un método. Comunica la intención, protege comportamiento crítico y permite optimizaciones del runtime.

**4. ¿Qué pasa si uso `new` en lugar de `override`?**
Ocultas el método en lugar de sobrescribirlo: si la variable es del tipo base, se ejecuta el método de la base.

-----

## Práctica

**Ejercicio 1.** Tienes `Auto`, `Bicicleta` y `Dron`. Los tres se pueden mover; `Auto` y `Dron` necesitan cargar energía; `Auto` y `Bicicleta` comparten la lógica de "cantidad de ruedas". ¿Qué interfaces y clases abstractas diseñarías?

<details>
<summary>Solución</summary>

```csharp
public interface IMovible { void Mover(); }
public interface IRecargable { void Recargar(); }

public abstract class VehiculoConRuedas : IMovible
{
    protected VehiculoConRuedas(int ruedas) => Ruedas = ruedas;
    public int Ruedas { get; }
    public abstract void Mover();
}

public sealed class Auto() : VehiculoConRuedas(4), IRecargable
{
    public override void Mover() { /* ... */ }
    public void Recargar() { /* ... */ }
}

public sealed class Bicicleta() : VehiculoConRuedas(2)
{
    public override void Mover() { /* ... */ }
}

public sealed class Dron : IMovible, IRecargable
{
    public void Mover() { /* ... */ }
    public void Recargar() { /* ... */ }
}
```

Moverse y recargar son **capacidades** (interfaces). Las ruedas son **estado compartido** entre tipos que sí "son" vehículos con ruedas (clase abstracta). `Dron` no hereda de esa base porque no tiene ruedas.

</details>

**Ejercicio 2.** ¿Qué imprime este código? ¿Por qué? ¿Cómo lo corregirías para que imprima "Guau"?

```csharp
public class Animal { public void Sonido() => Console.WriteLine("..."); }
public class Perro : Animal { public new void Sonido() => Console.WriteLine("Guau"); }

Animal a = new Perro();
a.Sonido();
```

<details>
<summary>Solución</summary>

Imprime `...`. `Sonido` no es `virtual`, y `Perro` lo **oculta** con `new`: la llamada se resuelve por el tipo de la variable (`Animal`), no por el del objeto.

```csharp
public class Animal { public virtual void Sonido() => Console.WriteLine("..."); }
public class Perro : Animal { public override void Sonido() => Console.WriteLine("Guau"); }
```

Con `virtual` y `override`, el runtime ejecuta el método del tipo real (`Perro`).

</details>

-----

## Siguiente lección

[Inyección de dependencias](06-inyeccion-de-dependencias.md)
