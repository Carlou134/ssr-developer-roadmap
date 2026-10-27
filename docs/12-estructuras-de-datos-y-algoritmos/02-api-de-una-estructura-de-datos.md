# API de una estructura de datos

## En una frase

La API de una estructura de datos es el conjunto de operaciones que ofrece hacia afuera, separada de cómo las resuelve por dentro, y **mientras esa API no cambie, puedes reemplazar la implementación sin romper a nadie**.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es una estructura de datos y la operación dominante: [Por qué estructuras de datos](01-por-que-estructuras-de-datos.md).
* Interfaces en C#: [Abstract vs interface](../02-calidad-de-codigo-y-arquitectura/05-abstract-vs-interface.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **API (*Application Programming Interface*):** las operaciones que un componente ofrece para que otro código lo use.
* **API pública:** la parte de la API pensada para quien usa el componente desde afuera.
* **Implementación:** cómo el componente resuelve esas operaciones por dentro.
* **Encapsulación:** ocultar la implementación para que solo se pueda usar la API pública.
* **Invariante:** regla que la estructura garantiza siempre, como "en una pila solo se saca el último elemento agregado".
* **Pila (*stack*):** estructura donde solo se agrega y se saca por arriba: el último en entrar es el primero en salir.

-----

## El problema

Necesitas registrar números a medida que llegan y después preguntar si un número ya llegó. La primera versión expone la lista directamente:

```csharp
public class NumberRegistry
{
    public List<double> Numbers = new();
}

// En otras partes del sistema:
registry.Numbers.Add(250);
var exists = registry.Numbers.Contains(17);
var first = registry.Numbers[0];
registry.Numbers.Insert(0, -42);
```

Un día llegan diez millones de números y `Contains` se vuelve lento, porque recorre toda la lista. La solución es obvia: usar un `HashSet<double>`. Pero cuando cambias el tipo, **todo el código que usaba `Numbers[0]` o `Insert` deja de compilar**, porque un `HashSet` no tiene posiciones.

Las consecuencias:

* un cambio interno de rendimiento se convierte en un cambio en decenas de archivos;
* cualquiera puede hacer `Numbers.Clear()` o meter datos sin validar, y la clase no puede impedirlo;
* el código que usa la clase depende de **cómo** guarda los datos, no de **qué** hace.

-----

## Cómo funciona

### 1. API pública frente a implementación

El usuario de la clase solo necesita dos operaciones: guardar un número y preguntar si existe. Esa es la API. Cómo se guardan los números es la implementación, y puede haber varias:

```csharp
// Implementación 1: lista. Contains recorre todo.
public class ListNumberRegistry
{
    private readonly List<double> _numbers = new();

    public void Store(double number) => _numbers.Add(number);
    public bool Contains(double number) => _numbers.Contains(number);
}

// Implementación 2: hash set. Contains va directo.
public class HashNumberRegistry
{
    private readonly HashSet<double> _numbers = new();

    public void Store(double number) => _numbers.Add(number);
    public bool Contains(double number) => _numbers.Contains(number);
}
```

Desde afuera son idénticas: `Store` y `Contains`. Por dentro, una revisa los elementos uno por uno y la otra salta directo. Cambiar de una a otra no toca ni una línea del código que las usa.

Lo mismo pasa con las colecciones que ya usas: llamas a `list.Add(x)` sin saber que por dentro, cuando el array se llena, se reserva uno más grande y se copia todo. **No necesitas conocer la implementación para usar la API.** Esa es justamente la idea.

### 2. Encapsulación: que la implementación no se pueda tocar

Separar la API no sirve de mucho si cualquiera puede saltársela. En C#, `private` lo garantiza el compilador:

```csharp
var registry = new HashNumberRegistry();
registry._numbers.Clear();
```

```text
error CS0122: 'HashNumberRegistry._numbers' is inaccessible due to its protection level
```

En JavaScript, durante años no hubo campos privados y se usaba la **convención** del guion bajo (`this._array`): una señal para otros desarrolladores, no una protección. Desde ES2022 existen los campos privados reales con `#`:

```javascript
class NumberRegistry {
  #numbers = new Set();

  store(num) { this.#numbers.add(num); }
  has(num) { return this.#numbers.has(num); }
}

const registry = new NumberRegistry();
registry.#numbers.clear();
```

```text
SyntaxError: Private field '#numbers' must be declared in an enclosing class
```

> **Ojo:** mucho material todavía dice que "JavaScript no soporta miembros privados". Es información anterior a ES2022. Hoy usa `#` y deja el `_` para código que no puedes migrar.

### 3. API restringida: ofrecer menos a propósito

Una pila es un caso clásico. Una `List<T>` ya permite agregar y sacar del final, así que podría servir de pila. Pero también permite insertar al principio, borrar del medio y leer cualquier posición, y todo eso **rompe la regla de la pila**.

Envolverla en una clase con una API reducida protege esa regla:

```csharp
public class ArrayStack<T>
{
    private readonly List<T> _items = new();

    public int Count => _items.Count;

    public void Push(T item) => _items.Add(item);

    public T Pop()
    {
        if (_items.Count == 0)
            throw new InvalidOperationException("Stack empty.");

        var top = _items[^1];
        _items.RemoveAt(_items.Count - 1);
        return top;
    }

    public T Peek() =>
        _items.Count == 0 ? throw new InvalidOperationException("Stack empty.") : _items[^1];
}
```

Quien use `ArrayStack<T>` **no puede** romper el invariante, porque las operaciones que lo romperían no existen. Restringir la API no es una limitación: es una garantía.

Si lo usas mal (saltarte la API, aquí en JS con la convención del guion bajo):

```javascript
class Stack {
  constructor() { this._array = []; }
  push(value) { this._array.push(value); }
  pop() { return this._array.pop(); }
}

const stack = new Stack();
stack.push('a');
stack._array.unshift('intruso'); // nadie lo impide
```

No hay error: el código corre y la pila queda en un estado que su diseño no contempla. Es peor que un error, porque falla en silencio.

### 4. La fuga por la puerta de atrás

Hacer el campo `private` no alcanza si después devuelves la misma referencia:

```csharp
public class ArrayStack<T>
{
    private readonly List<T> _items = new();
    public List<T> Items => _items;   // devuelve la lista real
}

stack.Items.Insert(0, value);         // compila y rompe la pila
```

Si alguien necesita leer el contenido, ofrécele una vista de solo lectura o una copia:

| Opción | Qué devuelve | Costo | Riesgo |
| --- | --- | --- | --- |
| `public IReadOnlyList<T> Items => _items;` | La misma lista, tipada como solo lectura | Ninguno | Se puede castear de vuelta a `List<T>` |
| `public IReadOnlyList<T> Items => _items.AsReadOnly();` | Un envoltorio que no permite modificar | Mínimo | Refleja los cambios posteriores |
| `public T[] ToArray() => _items.ToArray();` | Una copia independiente | Copia todos los elementos | Ninguno |

### 5. La interfaz como contrato

El paso siguiente es escribir la API como una interfaz. Así el código que la usa depende del contrato, no de una clase concreta:

```csharp
public interface INumberRegistry
{
    void Store(double number);
    bool Contains(double number);
}
```

Ahora `ListNumberRegistry` y `HashNumberRegistry` implementan `INumberRegistry`, y elegir una u otra se reduce a una línea en la inyección de dependencias. Es el mismo principio que ya viste en arquitectura (depender de abstracciones), aplicado a una estructura de datos.

> **Ojo:** cada método público es una **promesa** que tienes que mantener. Si agregas `GetAt(int index)` "por si acaso", ya no puedes pasar a un `HashSet`, porque un set no tiene posiciones. Una API chica te deja libertad para cambiar la implementación.

-----

## Ejemplo completo

El mismo registro del problema, con la API separada de la implementación.

`NumberRegistry.cs`:

```csharp
public interface INumberRegistry
{
    void Store(double number);
    bool Contains(double number);
    int Count { get; }
}

public sealed class HashNumberRegistry : INumberRegistry
{
    private readonly HashSet<double> _numbers = new();

    public int Count => _numbers.Count;

    public void Store(double number)
    {
        if (double.IsNaN(number))
            throw new ArgumentException("NaN cannot be stored.", nameof(number));

        _numbers.Add(number);
    }

    public bool Contains(double number) => _numbers.Contains(number);
}
```

`Program.cs`:

```csharp
INumberRegistry registry = new HashNumberRegistry();

foreach (var n in new[] { 1, 250, -42, 0.4, 17 })
    registry.Store(n);

Console.WriteLine(registry.Contains(17));    // True
Console.WriteLine(registry.Contains(3));     // False
Console.WriteLine(registry.Count);           // 5
```

Resultado: `True`, `False` y `5`.

Qué observar:

* El código de `Program.cs` solo conoce `INumberRegistry`. Si mañana necesitas guardar los números en orden de llegada, cambias la implementación y `Program.cs` no se entera.
* El `HashSet` es privado: nadie puede hacer `Clear()` ni saltarse la validación de `Store`.
* La API es mínima: tres miembros. No expone posiciones ni la colección interna, así que el `HashSet` se puede cambiar por cualquier otra estructura.
* `Store` valida antes de guardar. Eso solo es posible porque **toda** entrada de datos pasa por la API.

-----

## Errores comunes

**1. Exponer la colección interna como campo o propiedad pública.**
Qué pasa: un cambio de `List` a `HashSet` rompe la compilación en medio proyecto.
Por qué: los consumidores usaron métodos de la implementación (`[0]`, `Insert`) en lugar de una API propia.
Arreglo: campo `private` y métodos que expresen lo que el negocio necesita (`Store`, `Contains`).

**2. Getter que devuelve la referencia mutable.**
Qué pasa: alguien modifica la colección desde afuera y la clase queda en un estado inválido.
Por qué: `private` protege el campo, no el objeto al que apunta. Devolver `_items` le entrega la colección real a quien llama.
Arreglo: devuelve `AsReadOnly()`, una copia o un `IEnumerable<T>` (sección 4).

**3. Tratar el `_` de JavaScript como si fuera privado.**
Qué pasa: código externo usa `obj._array` y queda acoplado a la implementación.
Por qué: el guion bajo es una convención; el lenguaje no lo impide.
Arreglo: campos privados con `#` (ES2022).

**4. Usar un objeto de JavaScript como conjunto de números.**
Qué pasa: `store(1)` y después `has("1")` devuelve `true`.
Por qué: las claves de un objeto se convierten a string. `1` y `"1"` terminan siendo la misma clave.
Arreglo: usa `Set` (o `Map` si necesitas un valor asociado). Conserva el tipo y su API comunica la intención.

**5. Agregar métodos públicos "por si acaso".**
Qué pasa: más adelante no puedes cambiar la implementación sin romper a alguien.
Por qué: cada método público es un compromiso que limita qué estructuras sirven por dentro.
Arreglo: empieza con lo mínimo y agrega operaciones cuando un caso real las pida.

-----

## Evolución y versiones (opcional)

* **ES2015:** llega la sintaxis `class` a JavaScript, sin miembros privados. Se populariza la convención `_`. Llegan también `Map` y `Set`, que reemplazan al objeto como diccionario o conjunto.
* **ES2022:** campos y métodos privados reales con `#`.
* **C#:** `private` existe desde la primera versión. `IReadOnlyList<T>` llega con .NET Framework 4.5 (2012).

-----

## Cuándo sí y cuándo no

**Envuelve la estructura en una API propia cuando:** hay reglas que proteger (invariantes, validaciones), la colección se comparte entre partes del sistema o es probable que la implementación cambie por rendimiento.

**No lo hagas cuando:** es una variable local dentro de un método o un DTO que solo transporta datos. Ahí envolver una `List` en una clase solo agrega ruido; usa la colección directamente.

-----

## Resumen en 5 líneas

1. La API es **qué** ofrece una estructura; la implementación es **cómo** lo hace.
2. Con la misma API puedes cambiar la implementación (lista por hash set) sin romper a quien la usa.
3. La encapsulación (`private` en C#, `#` en JS) impide saltarse la API; el `_` es solo una convención.
4. Restringir la API protege los invariantes: una pila que no ofrece `Insert` no se puede romper.
5. No devuelvas la colección interna, y no agregues métodos públicos que no necesitas: cada uno es una promesa.

-----

## Para profundizar (opcional)

<details>
<summary>¿IEnumerable&lt;T&gt; es suficiente para proteger la colección?</summary>

Devolver `IEnumerable<T>` oculta los métodos de modificación, pero si devuelves `_items` directamente, quien llama puede hacer `(List<T>)registry.Items` y modificarla. Además, `IEnumerable<T>` no expone `Count`, y algunos consumidores terminan llamando a `.Count()` de LINQ o recorriéndola varias veces. Si quieres una protección real, usa `AsReadOnly()` o una copia; si solo quieres comunicar la intención, `IReadOnlyCollection<T>` suele ser el tipo más honesto.

</details>

<details>
<summary>¿Qué tiene que ver esto con los tipos de datos abstractos?</summary>

En teoría de computación se distingue el **tipo de dato abstracto** (TDA, o ADT en inglés), que define las operaciones y su comportamiento ("una pila tiene push, pop y peek, y pop devuelve el último agregado"), de la **estructura de datos** concreta que lo implementa (una pila sobre un array o sobre una lista enlazada). La API pública de esta lección es la forma práctica de expresar un TDA en código. Vas a ver esta distinción en cada estructura de las próximas lecciones.

</details>

-----

## En entrevista (opcional)

### Respuesta corta (junior)

La API es lo que una clase ofrece hacia afuera y la implementación es cómo lo resuelve por dentro. Se separan para poder cambiar la implementación sin afectar el código que la usa.

### Respuesta ampliada (semi-senior)

Diseño la API a partir de lo que necesita quien la consume, no de la estructura que uso por dentro. La mantengo mínima porque cada método público restringe las implementaciones posibles. Encapsulo la colección para proteger sus invariantes y sus validaciones, y no devuelvo referencias mutables. Cuando hay posibilidad real de cambiar la implementación, o la necesito reemplazar en tests, la expreso como interfaz y la inyecto. Para colecciones locales o DTO no envuelvo nada: sería sobreingeniería.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no usar directamente `Stack<T>` de .NET en lugar de escribir tu propia pila?**
En producción, sí: usa `Stack<T>`. Escribirla sirve para entender la separación entre API e implementación. Y la técnica aplica a tus propias clases de dominio, donde no existe una versión de la librería.

**2. Si hago el campo `private`, ¿ya está encapsulado?**
No necesariamente. Si una propiedad devuelve la misma referencia, la colección sigue siendo modificable desde afuera.

**3. ¿Cuál es la diferencia entre `_campo` y `#campo` en JavaScript?**
`_campo` es una convención: el campo es público y accesible. `#campo` es privado de verdad: acceder desde fuera de la clase es un error de sintaxis.

-----

## Práctica

**Ejercicio 1.** Diseña la API (solo las firmas públicas) de un `UniqueVisitorCounter` para un sitio web. Debe registrar visitas por Id de usuario y responder cuántos visitantes **distintos** hubo. Luego elige la implementación interna.

<details>
<summary>Solución</summary>

```csharp
public interface IUniqueVisitorCounter
{
    void RegisterVisit(Guid userId);
    int UniqueVisitors { get; }
}
```

Implementación: un `HashSet<Guid>` privado. `RegisterVisit` llama a `Add`, que ignora los duplicados, y `UniqueVisitors` devuelve `Count`. La API no expone la colección ni la lista de Ids: si mañana hay millones de visitantes y conviene un algoritmo aproximado como HyperLogLog, cambias la implementación sin tocar a los consumidores.

</details>

**Ejercicio 2.** ¿Qué tiene de malo esta clase?

```csharp
public class ShoppingCart
{
    public List<CartItem> Items { get; set; } = new();
    public decimal Total { get; set; }

    public void AddItem(CartItem item)
    {
        Items.Add(item);
        Total += item.Price * item.Quantity;
    }
}
```

<details>
<summary>Solución</summary>

El invariante es "`Total` es la suma de los ítems", y se puede romper de tres formas: `cart.Items.Add(...)` sin pasar por `AddItem`, `cart.Items = new()` y `cart.Total = 0`. Corrección:

```csharp
public class ShoppingCart
{
    private readonly List<CartItem> _items = new();

    public IReadOnlyList<CartItem> Items => _items.AsReadOnly();
    public decimal Total => _items.Sum(i => i.Price * i.Quantity);

    public void AddItem(CartItem item) => _items.Add(item);
}
```

Ahora la única forma de modificar el carrito es `AddItem`, y el total se calcula a partir de los ítems, así que nunca puede quedar desincronizado.

</details>

-----

## Siguiente lección

[Notación asintótica y Big O](03-notacion-asintotica.md)
