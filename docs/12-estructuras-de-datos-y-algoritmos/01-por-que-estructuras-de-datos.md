# Por qué estructuras de datos

## En una frase

Una estructura de datos es la forma en que organizas la información en memoria para guardarla y recuperarla, y **elegir la correcta decide si tu algoritmo salta directo al dato o tiene que revisarlos todos**.

-----

## Antes de empezar

Conviene que ya sepas:

* Usar colecciones básicas de C# (`List<T>`, `Dictionary<TKey, TValue>`) o sus equivalentes en JavaScript (arrays y objetos/`Map`).
* LINQ básico: `FirstOrDefault`, `Where`, `ToDictionary`.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Estructura de datos:** forma de organizar datos para guardarlos, mantenerlos y recuperarlos.
* **Algoritmo:** secuencia de pasos que transforma datos en un resultado.
* **Índice:** posición numérica de un elemento en una colección secuencial.
* **Clave (*key*):** valor con nombre que identifica un elemento, como un email o un Id.
* **Asignación estática de memoria:** el tamaño se fija al crear la estructura y no cambia.
* **Asignación dinámica de memoria:** la estructura pide más (o libera) memoria mientras el programa corre.
* **Capacidad (*capacity*):** cuántos elementos caben en la memoria ya reservada, frente a cuántos hay (`Count`).
* **Tiempo de ejecución (*runtime*):** cuánto tarda una operación según la cantidad de datos.

-----

## El problema

Tienes un servicio de login. Los usuarios están cargados en memoria en una `List<User>` y cada intento de login busca al usuario por email:

```csharp
public class UserService(IEnumerable<User> users)
{
    private readonly List<User> _users = users.ToList(); // 1.000.000 de usuarios

    public User? FindByEmail(string email) =>
        _users.FirstOrDefault(u => u.Email == email);
}
```

Funciona. Con 100 usuarios en desarrollo, nadie nota nada. Pero `FirstOrDefault` sobre una lista **compara uno por uno** desde el principio hasta encontrar el email. Si el usuario está al final, son un millón de comparaciones por cada login.

Las consecuencias:

* el tiempo de respuesta crece en proporción a la cantidad de usuarios: el doble de usuarios, el doble de lentitud;
* con muchos logins concurrentes, la CPU se va a pique haciendo comparaciones inútiles;
* el bug no aparece en los tests ni en desarrollo, aparece en producción.

Es el supermercado desordenado: la pizza congelada junto a los morrones y el cepillo de dientes junto a la leche. Todo está ahí, pero para encontrar algo tienes que recorrer todos los pasillos.

-----

## Cómo funciona

### 1. Datos + algoritmos

Todo programa se apoya en dos cosas: **datos** y **algoritmos** que los transforman. La estructura de datos es el puente entre ambos: define qué operaciones son baratas y cuáles son caras, y por lo tanto qué algoritmos puedes usar de forma eficiente.

```text
datos crudos ──► ESTRUCTURA DE DATOS ──► algoritmo ──► resultado
                 (cómo están organizados) (qué tan rápido puede trabajar)
```

El mismo algoritmo ("buscar un usuario por email") cuesta un millón de pasos en una lista y uno solo en un diccionario. El algoritmo no cambió: cambió la organización de los datos.

### 2. Las cuatro funciones de una estructura

Toda estructura de datos resuelve cuatro cosas. Al evaluarla, hazte estas preguntas:

| Función | Pregunta clave | En nuestro login |
| --- | --- | --- |
| **Ingresar** (*input*) | ¿Qué datos acepta? ¿Se agregan al inicio, al final o en el medio? ¿Se actualizan o eliminan? | Se registran usuarios nuevos; a veces se borra uno. |
| **Procesar** (*processing*) | ¿Qué tiene que reacomodar la estructura cuando algo se agrega, cambia o se elimina? | Si un usuario cambia su email, hay que actualizar la clave. |
| **Mantener** (*maintaining*) | ¿Qué relaciones entre datos se conservan? ¿Cuánta memoria hay que reservar? | Cada email apunta a un único usuario. |
| **Recuperar** (*retrieving*) | ¿Cómo vuelvo a obtener el dato? ¿Cuántos pasos cuesta? | **La operación dominante:** se ejecuta en cada login. |

> **Ojo:** casi nunca todas las operaciones importan igual. Busca la **operación dominante** (la que más se repite o la que está en el camino crítico) y elige la estructura que la haga barata. En el login es *recuperar por email*, no *agregar usuarios*.

### 3. Acceso por índice frente a acceso por clave

Las dos familias que ya usas a diario:

* **Secuenciales** (`T[]`, `List<T>`; arrays en JS): los elementos están en orden y se accede **por posición**.
* **Por clave** (`Dictionary<TKey, TValue>`; objetos o `Map` en JS): cada elemento se guarda y se recupera **por un nombre**.

```csharp
var names = new List<string> { "Ana", "Luis", "Marta" };
var second = names[1];                       // "Luis": salta directo a la posición 1

var ages = new Dictionary<string, int> { ["Ana"] = 30, ["Luis"] = 25 };
var luisAge = ages["Luis"];                  // 25: salta directo por la clave
var found = names.Contains("Marta");         // true, pero revisa uno por uno
```

Las dos saltan directo **si sabes lo que la estructura sabe**: la lista conoce posiciones, el diccionario conoce claves. Buscar por contenido en una lista (`Contains`, `FirstOrDefault`) obliga a recorrerla.

Si lo usas mal:

```csharp
var third = names[3];       // la lista tiene 3 elementos: índices 0, 1 y 2
var pedro = ages["Pedro"];  // la clave no existe
```

```text
System.ArgumentOutOfRangeException: Index was out of range. Must be non-negative and less than the size of the collection. (Parameter 'index')
System.Collections.Generic.KeyNotFoundException: The given key 'Pedro' was not present in the dictionary.
```

Cuando no sabes si la clave existe, usa `TryGetValue` o `GetValueOrDefault` en lugar del indexador.

### 4. Memoria: tamaño fijo frente a tamaño que crece

Algunas estructuras reservan un tamaño fijo al crearse; otras crecen mientras el programa corre.

```csharp
var fixedSize = new int[3];   // estática: 3 lugares, para siempre
fixedSize[3] = 10;            // no hay lugar 3
```

```text
System.IndexOutOfRangeException: Index was outside the bounds of the array.
```

```csharp
var growing = new List<int>();   // dinámica: crece sola
growing.Add(1);                  // por dentro hay un array; cuando se llena,
growing.Add(2);                  // reserva uno más grande (el doble) y copia todo
```

En C#, igual que en JavaScript o Python, el **recolector de basura** libera la memoria por ti. En C, en cambio, tú pides y liberas memoria a mano, y ahí la diferencia entre estático y dinámico es mucho más visible.

Que la memoria sea administrada no significa que no tengas control: si sabes cuántos elementos vas a cargar, reservar la capacidad evita copias intermedias.

```csharp
var users = new List<User>(1_000_000);   // reserva una vez, sin realocar al crecer
```

> **Ojo:** muchos materiales dicen que "las pilas y los arrays son estáticos" y que "los heaps y las listas enlazadas son dinámicos". Es una simplificación engañosa. Una **pila** es un *comportamiento* (último en entrar, primero en salir) y puede implementarse sobre un array fijo o sobre uno que crece: `Stack<T>` de .NET crece sola. Y el *heap* como estructura de datos (una cola de prioridad, `PriorityQueue<TElement, TPriority>`) no es lo mismo que el *heap* de memoria donde viven los objetos. Mismo nombre, conceptos distintos.

### 5. Cómo elegir la estructura

Tres preguntas, en este orden:

1. **¿Para qué son los datos?** ¿Cuál es la operación dominante: buscar, ordenar, recorrer, sacar el primero, sacar el último?
2. **¿Necesitas controlar la memoria?** Tamaño conocido o desconocido, cuánto puede crecer, si conviene reservar capacidad.
3. **¿Cuánto tarda cada estructura en esa operación?** Dos estructuras pueden resolver lo mismo y una ser miles de veces más rápida. Esto se mide con la notación asintótica, el tema de la próxima lección.

Guía rápida en .NET:

| Operación dominante | Estructura | Equivalente en JS |
| --- | --- | --- |
| Acceso por posición, recorrer en orden | `T[]`, `List<T>` | `Array` |
| Buscar por clave | `Dictionary<TKey, TValue>` | `Map`, objeto |
| Saber si algo existe (sin valor asociado) | `HashSet<T>` | `Set` |
| Último en entrar, primero en salir (deshacer) | `Stack<T>` | `Array` con `push`/`pop` |
| Primero en entrar, primero en salir (cola de tareas) | `Queue<T>` | `Array` con `push`/`shift` |
| Sacar siempre el de mayor prioridad | `PriorityQueue<TElement, TPriority>` | No hay nativa |
| Mantener los datos ordenados por clave | `SortedDictionary<TKey, TValue>`, `SortedSet<T>` | No hay nativa |

-----

## Ejemplo completo

Medimos el mismo problema del login con las dos estructuras: mil búsquedas sobre un millón de usuarios.

`Program.cs` (proyecto de consola, `dotnet new console`):

```csharp
using System.Diagnostics;

const int TotalUsers = 1_000_000;
const int Lookups = 1_000;

var users = new List<User>(TotalUsers);
for (var i = 0; i < TotalUsers; i++)
    users.Add(new User(i, $"user{i}@mail.com"));

// Los emails del final de la lista: el peor caso para la búsqueda secuencial
var emailsToFind = Enumerable.Range(TotalUsers - Lookups, Lookups)
    .Select(i => $"user{i}@mail.com")
    .ToArray();

var sw = Stopwatch.StartNew();
foreach (var email in emailsToFind)
    _ = users.FirstOrDefault(u => u.Email == email);
sw.Stop();
Console.WriteLine($"List:       {sw.ElapsedMilliseconds} ms");

sw.Restart();
var usersByEmail = users.ToDictionary(u => u.Email, StringComparer.OrdinalIgnoreCase);
sw.Stop();
Console.WriteLine($"Construir:  {sw.ElapsedMilliseconds} ms (una sola vez)");

sw.Restart();
foreach (var email in emailsToFind)
    _ = usersByEmail.GetValueOrDefault(email);
sw.Stop();
Console.WriteLine($"Dictionary: {sw.ElapsedMilliseconds} ms");

record User(int Id, string Email);
```

Resultado: la versión con `List` tarda **varios segundos**; la versión con `Dictionary`, prácticamente **0 ms**. Construir el diccionario cuesta unos cientos de milisegundos, pero se paga una sola vez. Los números exactos dependen de tu máquina; la diferencia de órdenes de magnitud, no.

El servicio corregido:

```csharp
public class UserService(IEnumerable<User> users)
{
    private readonly Dictionary<string, User> _usersByEmail =
        users.ToDictionary(u => u.Email, StringComparer.OrdinalIgnoreCase);

    public User? FindByEmail(string email) =>
        _usersByEmail.GetValueOrDefault(email);
}
```

Qué observar:

* Se eligió la estructura según la **operación dominante** (recuperar por email), no según la costumbre.
* El diccionario se construye **una vez** en el constructor, no en cada llamada.
* `StringComparer.OrdinalIgnoreCase` mantiene una regla del negocio dentro de la estructura: `Ana@Mail.com` y `ana@mail.com` son el mismo usuario.
* `GetValueOrDefault` evita la `KeyNotFoundException` cuando el email no existe, que en un login es un caso normal, no excepcional.

-----

## Errores comunes

**1. Buscar en una lista dentro de un bucle.**
Qué pasa: un reporte que procesa 10.000 pedidos tarda minutos.
Por qué: `orders.Select(o => customers.First(c => c.Id == o.CustomerId))` recorre la lista de clientes **por cada pedido**: 10.000 × 10.000 comparaciones.
Arreglo: construye antes `customers.ToDictionary(c => c.Id)` y busca por clave dentro del bucle.

**2. Construir la estructura en cada llamada.**
Qué pasa: cambiaste a `Dictionary` y el código quedó igual de lento o peor.
Por qué: `ToDictionary` dentro del método recorre toda la colección en cada llamada. Pagaste el costo de construir sin aprovechar las búsquedas rápidas.
Arreglo: constrúyela una vez (constructor, caché, campo) y reutilízala.

**3. Esperar orden de un `Dictionary`.**
Qué pasa: al recorrerlo, los elementos salen en un orden distinto al esperado, o el orden cambia después de eliminar elementos.
Por qué: un `Dictionary` está organizado para buscar por clave, no para mantener un orden. El orden de enumeración no está garantizado.
Arreglo: si necesitas orden por clave, usa `SortedDictionary`; si necesitas el orden de inserción, guarda también una `List`.

**4. Usar `List<T>` para todo por costumbre.**
Qué pasa: `Contains`, `Remove` y `FirstOrDefault` en caminos críticos que se vuelven lentos con volumen.
Por qué: la lista solo es rápida por posición; todo lo demás la recorre.
Arreglo: antes de declarar la colección, pregúntate cuál es la operación dominante (sección 5).

-----

## Cuándo sí y cuándo no

**Piensa la estructura con cuidado cuando:** el volumen de datos crece en producción, la operación se repite mucho (por request, dentro de bucles) o está en el camino crítico (login, checkout, búsquedas).

**No te obsesiones cuando:** la colección tiene diez elementos, el código corre una sola vez al iniciar o la claridad pesa más que microsegundos. Ahí elige lo más legible. Y si dudas, **mide** con `Stopwatch` o BenchmarkDotNet antes de optimizar.

-----

## Resumen en 5 líneas

1. Una estructura de datos organiza la información para que los algoritmos trabajen rápido sobre ella.
2. Toda estructura resuelve cuatro funciones: ingresar, procesar, mantener y recuperar.
3. Elige según la operación dominante: posición → lista, clave → diccionario, existencia → set.
4. Estático o dinámico depende de la implementación, no del nombre; en C# el GC libera memoria, pero tú puedes reservar capacidad.
5. Dos estructuras pueden dar el mismo resultado con tiempos que difieren en órdenes de magnitud.

-----

## Para profundizar (opcional)

<details>
<summary>¿Por qué un Dictionary encuentra la clave sin recorrer?</summary>

Calcula un número a partir de la clave (su *hash*, con `GetHashCode`) y lo usa para saber en qué "casillero" (*bucket*) de un array interno está guardado el elemento. Va directo a ese casillero y compara solo los pocos elementos que comparten casillero. Por eso, si cambias la propiedad que se usa como clave después de insertar el objeto, el diccionario lo busca en el casillero equivocado y no lo encuentra. Las tablas hash tienen su propia lección más adelante.

</details>

<details>
<summary>Si List&lt;T&gt; copia todo al crecer, ¿no es lenta para agregar?</summary>

Cada vez que se llena, reserva el doble de capacidad y copia los elementos. Esa copia es cara, pero ocurre cada vez menos seguido: después de llenar 4 lugares copia 4, después de 8 copia 8, después de 16 copia 16. Repartido entre todos los `Add`, el costo promedio por elemento es constante. Se llama **costo amortizado** y lo verás formalmente con la notación asintótica.

</details>

-----

## En entrevista (opcional)

### Respuesta corta (junior)

Una estructura de datos es la forma de organizar datos para guardarlos y recuperarlos. Hay que elegirla según lo que más se va a hacer con los datos: por ejemplo, un diccionario si se busca por clave y una lista si se accede por posición.

### Respuesta ampliada (semi-senior)

Primero identifico la operación dominante y el volumen esperado en producción, no en desarrollo. Con eso elijo la estructura que hace barata esa operación, aceptando que otras se vuelvan más caras: un `Dictionary` acelera la búsqueda por clave a cambio de más memoria y de perder el orden. También considero cuándo se construye la estructura (una vez frente a cada request) y si conviene reservar capacidad. Y si la colección es chica, priorizo la legibilidad y mido antes de optimizar.

### Preguntas frecuentes de seguimiento

**1. ¿Cuándo usarías un HashSet en lugar de una List?**
Cuando la operación dominante es preguntar "¿esto ya existe?": `HashSet.Contains` va directo por hash, `List.Contains` recorre todo. Además, el set evita duplicados por diseño.

**2. ¿Qué pierdes al pasar de List a Dictionary?**
Memoria extra por la tabla interna, el orden garantizado y el acceso por posición. Además, la clave tiene que ser única.

**3. ¿Un array y una List son lo mismo?**
No. El array tiene tamaño fijo; la `List<T>` envuelve un array y lo reemplaza por uno más grande cuando se llena.

-----

## Práctica

**Ejercicio 1.** Elige la estructura de .NET para cada caso y justifica con la operación dominante:

1. El botón "deshacer" de un editor de texto.
2. Validar en el registro si un nombre de usuario ya está tomado.
3. Una cola de correos para enviar en el orden en que llegaron.
4. Obtener el carrito de compras a partir del Id del usuario.

<details>
<summary>Solución</summary>

1. `Stack<T>`: deshacer siempre revierte **la última** acción (último en entrar, primero en salir).
2. `HashSet<string>` con `StringComparer.OrdinalIgnoreCase`: solo importa si existe, no hay valor asociado.
3. `Queue<T>`: el primero que llegó es el primero que sale.
4. `Dictionary<Guid, Cart>`: búsqueda por clave.

</details>

**Ejercicio 2.** ¿Qué tiene de malo este código? Hay 50.000 productos y 200.000 ítems de pedidos.

```csharp
public decimal CalculateTotal(List<OrderItem> items, List<Product> products)
{
    decimal total = 0;
    foreach (var item in items)
    {
        var product = products.First(p => p.Id == item.ProductId);
        total += product.Price * item.Quantity;
    }
    return total;
}
```

<details>
<summary>Solución</summary>

`products.First` recorre la lista de productos por cada ítem: hasta 200.000 × 50.000 = 10.000 millones de comparaciones. La operación dominante es "buscar producto por Id", así que corresponde un diccionario construido una sola vez antes del bucle:

```csharp
public decimal CalculateTotal(List<OrderItem> items, List<Product> products)
{
    var productsById = products.ToDictionary(p => p.Id);

    decimal total = 0;
    foreach (var item in items)
        total += productsById[item.ProductId].Price * item.Quantity;
    return total;
}
```

Ahora son 50.000 pasos para construir más 200.000 búsquedas directas. Aquí el indexador `[]` es correcto: si un ítem referencia un producto inexistente, los datos están corruptos y la excepción debe saltar.

</details>

-----

## Siguiente lección

[API de una estructura de datos](02-api-de-una-estructura-de-datos.md)
