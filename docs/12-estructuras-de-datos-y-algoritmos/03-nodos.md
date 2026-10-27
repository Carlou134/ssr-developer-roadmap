# Nodos

## En una frase

Un nodo es una pieza que guarda **un dato y uno o más enlaces a otros nodos**, y es el bloque con el que se construyen las listas enlazadas, las pilas, las colas, los árboles y los grafos.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es una estructura de datos y qué es una API: [API de una estructura de datos](02-api-de-una-estructura-de-datos.md).
* La diferencia entre tipos por valor y por referencia en C# (`struct` frente a `class`).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Nodo (*node*):** objeto que contiene un dato y uno o más enlaces a otros nodos.
* **Enlace (*link*, *pointer*):** referencia de un nodo a otro. Si vale `null`, ese camino termina ahí.
* **Nodo huérfano (*orphaned node*):** nodo al que ya no apunta ningún enlace, así que el programa no lo puede alcanzar.
* **Lista enlazada (*linked list*):** cadena de nodos en la que cada uno apunta al siguiente.

-----

## El problema

Tienes tres nodos encadenados y quieres sacar el del medio:

```text
node_a ("duck") → node_b ("duck") → node_c ("goose") → null
```

La primera idea es "cortar" el enlace hacia `node_b`:

```csharp
nodeA.Next = null; // quería sacar node_b
```

Pero el resultado es este:

```text
node_a ("duck") → null

node_b ("duck") → node_c ("goose") → null   ← nadie apunta a estos
```

Las consecuencias:

* `node_b` salió de la cadena, pero `node_c` también, aunque no querías borrarlo;
* el único camino hasta `node_c` pasaba por `node_b`, así que los dos quedaron huérfanos y sus datos se perdieron para el programa.

-----

## Cómo funciona

### 1. Un nodo: dato + enlace

Un nodo tiene dos partes: el dato que guarda y el enlace al siguiente nodo. En C# el enlace es una referencia a otro objeto del mismo tipo:

```csharp
public class Node<T>
{
    public T Data { get; }
    public Node<T>? Next { get; set; }

    public Node(T data, Node<T>? next = null)
    {
        Data = data;
        Next = next;
    }
}
```

El dato puede ser de cualquier tipo: un número (`5`), un texto (`"five"`), un decimal (`5.1m`), un arreglo (`[5, 3, 4]`) o incluso `null`. Por eso se usa un genérico `T`.

En otros lenguajes, como C, el enlace es un **puntero**: una dirección de memoria. En C# es una **referencia** y el runtime administra la memoria por ti, pero la idea es la misma: el enlace "apunta" a otro nodo.

### 2. `null` marca el final del camino

Si un enlace vale `null`, ya no hay a dónde seguir. Así se recorre una cadena de nodos:

```csharp
for (var current = head; current is not null; current = current.Next)
    Console.Write($"{current.Data} → ");

Console.WriteLine("null");
```

Si lo usas mal, saltándote la comprobación de `null`:

```csharp
var third = nodeA.Next.Next.Next; // node_c.Next es null
Console.WriteLine(third.Data);
```

```text
warning CS8602: Dereference of a possibly null reference.
System.NullReferenceException: Object reference not set to an instance of an object.
```

Con `<Nullable>enable</Nullable>` el compilador te avisa con CS8602 antes de ejecutar. Si ignoras la advertencia, el programa falla en tiempo de ejecución.

### 3. Un nodo puede tener varios enlaces

Cada estructura decide cuántos enlaces tiene un nodo y hacia dónde apuntan:

```text
node_a → node_b          un enlace: lista enlazada simple

     node_c              dos enlaces hacia abajo: árbol binario
     ↙    ↘
 node_d  node_e

node_f ⇄ node_g          enlaces en los dos sentidos: lista doblemente enlazada
```

| Enlaces del nodo | Estructura típica | Para qué sirve |
| --- | --- | --- |
| `Next` | Lista enlazada, pila, cola | Recorrer en un solo sentido |
| `Next` y `Previous` | Lista doblemente enlazada | Recorrer hacia adelante y hacia atrás |
| `Left` y `Right` | Árbol binario | Dividir los datos en dos ramas |
| Lista de vecinos | Grafo | Relaciones de muchos a muchos |

Lo que cambia entre una estructura y otra no es el nodo en sí, sino **las reglas que la estructura impone sobre sus enlaces**.

> **Ojo:** con enlaces en los dos sentidos (`node_f ⇄ node_g`), si recorres siguiendo cualquier enlace sin control, vas y vuelves para siempre. Cada recorrido tiene que seguir una dirección: solo `Next` o solo `Previous`.

### 4. Nodos huérfanos y cómo sacar un nodo sin perder el resto

En muchas estructuras, a cada nodo solo apunta **un** enlace. Si ese enlace se pierde, el nodo y todos los que cuelgan de él quedan huérfanos.

Para sacar `node_b` sin perder `node_c`, no cortes el enlace: **redirígelo**. Haz que `node_a` apunte a lo que apuntaba `node_b`:

```csharp
nodeA.Next = nodeB.Next; // node_a ahora apunta a node_c
```

```text
Antes:   node_a → node_b → node_c → null
Después: node_a ─────────→ node_c → null
```

El orden importa: **primero lees `nodeB.Next`, después cambias `nodeA.Next`**. Si lo haces al revés, ya perdiste el camino hasta `node_c`.

### 5. ¿Hay que borrar el nodo que sacaste?

En C#, no. Cuando `node_a` deja de apuntar a `node_b`, nada del programa llega a `node_b`, y el **recolector de basura** (GC) libera esa memoria por su cuenta.

Que `node_b` todavía apunte a `node_c` no importa. Lo que decide si un objeto sigue vivo es si **alguien apunta a él**, no a quién apunta él.

| Lenguaje | Qué pasa con el nodo que sacaste |
| --- | --- |
| C#, Java, JavaScript, Python | El GC lo libera cuando nadie lo alcanza |
| C | Tienes que llamar a `free`; si no, la memoria queda ocupada (*memory leak*) |
| C++ | `delete` manual o punteros inteligentes como `std::unique_ptr` |

> **Ojo:** si otra variable todavía referencia a `node_b`, por ejemplo una variable local o un caché, el GC no lo libera. Sale de la cadena, pero sigue vivo en memoria.

-----

## Ejemplo completo

`AlgorithmsLab/Lesson03_Nodes/Exercise00.cs`:

```csharp
namespace AlgorithmsLab.Lesson03_Nodes
{
    public class Node<T>
    {
        public T Data { get; }
        public Node<T>? Next { get; set; }

        public Node(T data, Node<T>? next = null)
        {
            Data = data;
            Next = next;
        }
    }

    public static class Exercise00
    {
        public static void Run()
        {
            var nodeC = new Node<string>("goose");
            var nodeB = new Node<string>("duck", nodeC);
            var nodeA = new Node<string>("duck", nodeB);

            Print("Antes", nodeA);

            // Sacar node_b sin perder node_c: redirigir, no cortar.
            nodeA.Next = nodeB.Next;

            Print("Después", nodeA);
        }

        private static void Print<T>(string label, Node<T>? head)
        {
            Console.Write($"{label}: ");
            for (var current = head; current is not null; current = current.Next)
                Console.Write($"{current.Data} → ");
            Console.WriteLine("null");
        }
    }
}
```

Resultado:

```text
Antes: duck → duck → goose → null
Después: duck → goose → null
```

Qué observar:

* los nodos se crean de atrás hacia adelante para poder pasarle a cada uno su siguiente en el constructor;
* `Data` no tiene `set`: el dato no cambia, solo cambian los enlaces;
* no hay ninguna línea que "borre" `node_b`: el GC se encarga cuando la variable `nodeB` deja de usarse.

-----

## Errores comunes

**1. Cortar un enlace en lugar de redirigirlo.**
Qué pasa: desaparecen de la cadena más nodos de los que querías sacar.
Por qué: el enlace que cortaste era el único camino hasta los nodos siguientes, y quedaron huérfanos.
Arreglo: haz que el nodo anterior apunte al siguiente del que sacas: `previous.Next = removed.Next`.

**2. Cambiar los enlaces en el orden equivocado.**
Qué pasa: al insertar o sacar un nodo, el resto de la cadena se pierde.
Por qué: sobrescribiste un enlace antes de guardar a dónde apuntaba.
Arreglo: antes de sobrescribir un enlace, guarda o usa primero el valor que tenía. Para insertar `x` entre `a` y `b`: primero `x.Next = a.Next`, después `a.Next = x`.

**3. Seguir enlaces sin comprobar `null`.**
Qué pasa: `NullReferenceException` al llegar al final de la cadena.
Por qué: el último nodo tiene `Next == null`, y acceder a `.Data` o `.Next` de `null` falla.
Arreglo: recorre con `current is not null` como condición, o usa `?.` cuando un enlace puede faltar.

**4. Recorrer sin fin una estructura con enlaces en los dos sentidos.**
Qué pasa: el programa se queda en un bucle infinito.
Por qué: seguiste `Next` y `Previous` indistintamente, y `node_f → node_g → node_f → ...` nunca llega a `null`.
Arreglo: cada recorrido sigue un solo tipo de enlace. En grafos, donde los ciclos son normales, lleva un `HashSet` con los nodos ya visitados.

-----

## Cuándo sí y cuándo no

**Úsalo cuando:** quieres entender cómo funcionan por dentro las listas enlazadas, los árboles y los grafos, o cuando modelas un dominio que ya es una cadena o un árbol, como un organigrama, un árbol de categorías o los pasos de un flujo.

**No lo uses cuando:** solo necesitas una colección. En producción usa `List<T>`, `Queue<T>`, `Stack<T>` o `LinkedList<T>`, que ya están probadas y optimizadas. Escribir tu propio `Node<T>` para guardar una lista de productos es reinventar la rueda.

-----

## Resumen en 5 líneas

1. Un nodo guarda un dato y uno o más enlaces a otros nodos.
2. Un enlace en `null` marca el final del camino.
3. Cada estructura impone sus propias reglas sobre los enlaces: uno (lista), dos hacia abajo (árbol), en los dos sentidos (lista doble).
4. Si se pierde el único enlace hacia un nodo, ese nodo y todo lo que cuelga de él quedan huérfanos.
5. Para sacar un nodo, redirige el enlace del anterior al siguiente; en C# el GC libera el nodo que sacaste.

-----

## Para profundizar (opcional)

<details>
<summary>¿Por qué <code>Node&lt;T&gt;</code> es una <code>class</code> y no un <code>struct</code>?</summary>

Un `struct` guarda sus campos **dentro** de sí mismo. Si un `struct Node` tuviera un campo `Node Next`, ese campo tendría otro `Node` adentro, que tendría otro adentro, y así sin fin: su tamaño sería infinito. El compilador lo rechaza:

```csharp
public struct Node
{
    public int Data;
    public Node Next;
}
```

```text
error CS0523: Struct member 'Node.Next' of type 'Node' causes a cycle in the struct layout
```

Una `class` guarda solo una **referencia** al otro objeto, que ocupa siempre lo mismo (8 bytes en 64 bits), así que puede apuntar a otro objeto de su mismo tipo sin problema.

</details>

<details>
<summary>¿Qué diferencia hay entre un arreglo y una cadena de nodos en memoria?</summary>

Un arreglo ocupa un bloque **contiguo** de memoria: el elemento `i` está en una posición que se calcula directamente, así que acceder por índice es inmediato.

Los nodos están **dispersos** por la memoria y se conectan solo por sus enlaces. Para llegar al quinto nodo tienes que pasar por los cuatro anteriores. A cambio, insertar o sacar un nodo solo cambia uno o dos enlaces, sin mover los demás elementos, como pasaría en un arreglo.

</details>

-----

## En entrevista (opcional)

### Respuesta corta (junior)

Un nodo es un objeto con un dato y un enlace a otro nodo. Encadenando nodos se arman listas enlazadas, pilas, colas y árboles. Si un enlace es `null`, se llegó al final.

### Respuesta ampliada (semi-senior)

El nodo es la unidad básica de las estructuras enlazadas: lo que las distingue son las reglas sobre los enlaces, no el nodo. Su ventaja frente a un arreglo es que insertar o sacar solo cambia enlaces, sin mover elementos. Su costo es que no hay acceso directo por posición y cada nodo ocupa memoria extra por sus enlaces. Al modificarlos hay que cuidar el orden de las asignaciones para no dejar nodos huérfanos. En C# no se libera memoria a mano: el GC recoge los nodos que ya no son alcanzables.

### Preguntas frecuentes de seguimiento

**1. ¿Hay que borrar el nodo que sacaste de la cadena?**
En C#, no: cuando nada lo referencia, el GC lo libera. En C o C++ sí, o tienes una fuga de memoria.

**2. ¿Qué diferencia hay entre un nodo y una lista enlazada?**
El nodo es la pieza; la lista enlazada es la estructura que organiza nodos con reglas (una cabeza, un enlace al siguiente) y ofrece una API para agregar, sacar y recorrer.

**3. ¿Por qué un nodo no puede ser un `struct` en C#?**
Porque un `struct` que se contiene a sí mismo tendría tamaño infinito. El compilador lo rechaza con el error CS0523.

-----

## Práctica

**Ejercicio 1.** Tienes la cadena `a → b → c → d → null`. Escribe el código para insertar un nodo `x` entre `a` y `b`, y después sacar `c` sin perder `d`. ¿En qué orden van las asignaciones?

<details>
<summary>Solución</summary>

```csharp
// Insertar x entre a y b: primero x se engancha a b, después a apunta a x.
x.Next = a.Next;
a.Next = x;

// Sacar c: b pasa a apuntar a lo que apuntaba c.
b.Next = c.Next;
```

Resultado: `a → x → b → d → null`.

Si al insertar haces primero `a.Next = x`, ya no tienes forma de llegar a `b` desde `a`, y `x.Next = a.Next` haría que `x` se apunte a sí mismo: un ciclo infinito.

</details>

**Ejercicio 2.** ¿Qué tiene de malo este código? Se quiere sacar `nodeB` de la cadena `nodeA → nodeB → nodeC → null`.

```csharp
nodeB.Next = null;
nodeA.Next = nodeB.Next;
```

<details>
<summary>Solución</summary>

Las asignaciones están en el orden equivocado. La primera línea corta el único enlace hacia `nodeC`, así que cuando la segunda lee `nodeB.Next`, ya vale `null`. El resultado es `nodeA → null`: `nodeC` quedó huérfano.

Basta con una sola línea:

```csharp
nodeA.Next = nodeB.Next;
```

No hace falta poner `nodeB.Next = null`. Si nadie apunta a `nodeB`, el GC lo libera igual.

</details>

-----

## Siguiente lección

[Notación asintótica y Big O](04-notacion-asintotica.md)
