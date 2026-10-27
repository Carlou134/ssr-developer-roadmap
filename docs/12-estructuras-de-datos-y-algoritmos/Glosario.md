# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Algoritmo.** Secuencia de pasos que transforma datos en un resultado.

**API (*Application Programming Interface*).** Las operaciones que un componente ofrece para que otro código lo use, sin conocer cómo funciona por dentro.

**API pública.** La parte de la API pensada para quien usa el componente desde afuera. Cada método público es un compromiso que hay que mantener.

**Asignación dinámica de memoria.** La estructura pide más memoria (o la libera) mientras el programa corre. Ejemplo: `List<T>` al crecer.

**Asignación estática de memoria.** El tamaño se fija al crear la estructura y no cambia. Ejemplo: `new int[3]`.

**Campo privado.** Miembro de una clase que solo se puede usar desde dentro de ella. En C# se declara con `private`; en JavaScript, desde ES2022, con `#`.

**Capacidad (*capacity*).** Cuántos elementos caben en la memoria ya reservada. En `List<T>` puede ser mayor que `Count`, que es cuántos hay realmente.

**Clave (*key*).** Valor que identifica un elemento dentro de una estructura, como un email o un Id. Debe ser única en un diccionario.

**Contrato.** La API expresada formalmente, por ejemplo como una interfaz de C#. Quien la usa depende del contrato, no de la clase concreta.

**Costo amortizado.** Costo promedio de una operación repartido entre muchas ejecuciones, aunque algunas puntuales sean caras.

**Encapsulación.** Ocultar la implementación de un componente para que solo se pueda usar a través de su API pública.

**Enlace (*link*, *pointer*).** Referencia de un nodo a otro. Si vale `null`, ese camino termina ahí. En C es una dirección de memoria (puntero); en C#, una referencia administrada por el runtime.

**Estructura de datos.** Forma de organizar datos para ingresarlos, procesarlos, mantenerlos y recuperarlos.

**Hash.** Número calculado a partir de una clave que indica dónde guardarla dentro de una tabla. En .NET lo devuelve `GetHashCode`.

**Implementación.** Cómo un componente resuelve por dentro las operaciones de su API. Puede cambiar sin afectar a quien lo usa.

**Índice.** Posición numérica de un elemento en una colección secuencial. En C# empieza en 0.

**Invariante.** Regla que una estructura garantiza siempre. Por ejemplo, "en una pila solo se saca el último elemento agregado" o "el total del carrito es la suma de sus ítems".

**Lista enlazada (*linked list*).** Cadena de nodos en la que cada uno apunta al siguiente. El último apunta a `null`.

**Nodo (*node*).** Objeto que contiene un dato y uno o más enlaces a otros nodos. Es la pieza básica de listas enlazadas, árboles y grafos.

**Nodo huérfano (*orphaned node*).** Nodo al que ya no apunta ningún enlace, así que el programa no lo puede alcanzar. En C# el GC lo termina liberando.

**Operación dominante.** La operación que más se repite o la que está en el camino crítico. Es la que debe guiar la elección de la estructura.

**Pila (*stack*).** Estructura en la que solo se agrega (*push*) y se saca (*pop*) por arriba: el último en entrar es el primero en salir (LIFO).

**Recolector de basura (*garbage collector*, GC).** Parte del runtime que libera automáticamente la memoria de los objetos que ya no se usan.

**Tiempo de ejecución (*runtime*).** Cuánto tarda una operación según la cantidad de datos que procesa.
