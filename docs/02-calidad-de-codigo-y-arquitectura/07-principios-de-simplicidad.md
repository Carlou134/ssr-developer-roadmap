# DRY, KISS y YAGNI

Tres principios que apuntan al mismo lugar desde ángulos distintos: no agregar complejidad que no hace falta.

## DRY — Don't Repeat Yourself

Evitar duplicar la misma lógica en más de un lugar. Si una regla de negocio (como calcular un descuento) está copiada en 3 archivos distintos, el día que esa regla cambie hay que acordarse de actualizarla en los 3 lugares — y es cuestión de tiempo hasta que alguien se olvide de uno.

No significa eliminar toda repetición a cualquier costo — dos fragmentos de código que se ven parecidos hoy pero representan conceptos distintos del negocio pueden divergir mañana. DRY aplica a lógica que representa el MISMO conocimiento, no a coincidencias superficiales de código.

## DRY vs. la abstracción incorrecta

Hay una regla de oro de **Sandi Metz** (referente en diseño de software orientado a objetos) que resume esta tensión mejor que cualquier definición: **"la duplicación es mucho más barata que la abstracción incorrecta."**

Unir prematuramente dos procesos que hoy comparten 3 líneas puede terminar en una función gigante llena de flags booleanos (`esContratista`, `esEdicion`, `esVenta`) tratando de acomodar casos que en el fondo son distintos. Cuando uno de los dos procesos cambia por una razón de negocio que no afecta al otro, esa abstracción compartida se convierte en un cuello de botella — cualquier cambio arriesga romper algo que no tenía nada que ver.

**El criterio real para decidir si duplicar o unificar no es "¿se ven parecidos?", es "¿van a cambiar por la misma razón?"** Si dos cálculos hoy hacen lo mismo pero responden a reglas de negocio independientes (ej: el pago de un empleado de planta y el de un contratista, cada uno regido por normas distintas), hay que mantenerlos separados aunque compartan una línea — unificarlos es la "abstracción incorrecta" de la que habla Metz.

**La Regla de Tres:** no extraigas código repetido hacia una función o clase compartida la primera vez que lo ves duplicado — ni siquiera la segunda. Esperá a verlo en **al menos 3 lugares distintos**. Recién ahí el patrón real de cómo varía ese código se hace visible, y se puede diseñar la abstracción correcta en vez de adivinarla con dos únicos casos de referencia.

## KISS — Keep It Simple, Stupid

Resolver el problema con la solución más simple que funcione, sin sobrecomplicar. También se lo conoce como "Keep It Short and Simple", una variante más amable del mismo principio.

```csharp
// Sobrecomplicado
if (user != null && user.IsActive == true)
{
    // ...
}

// Simple, mismo resultado
if (user?.IsActive == true)
{
    // ...
}
```

**Sobre-arquitectura** es la violación típica de KISS: diseñar una solución demasiado grande o compleja para un problema que en realidad es simple. Conecta directo con [Simplicidad vs Flexibilidad](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md), ya visto en punto 1.

**Legibilidad vs. brevedad — no son lo mismo.** Comprimir todo en una sola línea no es más simple, es solo más corto:

```csharp
// Compacto, pero hay que leerlo de un tirón para entenderlo
public decimal CalcularValorInventario(List<Producto> inventario) =>
    inventario.Where(p => p.Activo && p.Stock > 50).Sum(p => (p.Precio * 0.90m) * p.Stock);

// Desglosado en pasos con nombres — el compilador lo optimiza igual, pero se lee y se debuggea mucho más fácil
public decimal CalcularValorInventario(List<Producto> inventario)
{
    var productosAptos = inventario.Where(p => p.Activo && p.Stock > 50).ToList();

    decimal valorTotal = 0m;
    foreach (var producto in productosAptos)
    {
        decimal precioConDescuento = producto.Precio * 0.90m;
        valorTotal += precioConDescuento * producto.Stock;
    }

    return valorTotal;
}
```

El código se lee muchas más veces de las que se escribe — un diseño compacto pero indescifrable no es un buen diseño. Con la segunda versión, si algo calcula mal, se puede poner un breakpoint exacto en `precioConDescuento` — algo imposible en la versión de una sola línea.

## YAGNI — You Aren't Gonna Need It

No implementar algo que no se necesita HOY, solo porque "por las dudas capaz hace falta después".

**Ejemplo de violación:** el cliente pidió solo "guardar usuarios", pero se termina construyendo 5 tipos de pago, un sistema de reportes y exportación a Excel — nada de eso fue pedido, y buena parte probablemente nunca se use.

Es, en esencia, el mismo principio que **No Big Design Up Front**, ya documentado en el archivo de trade-offs de punto 1: la mitad de lo que se anticipa nunca ocurre, y la otra mitad termina siendo distinta a como se imaginó.

## La diferencia entre los tres

- **DRY** → no repetir el mismo conocimiento en varios lugares.
- **KISS** → resolver con la solución más simple posible.
- **YAGNI** → no construir hoy lo que no se pidió.

Los tres apuntan a mantener la complejidad del sistema lo más baja posible — el primer trade-off documentado en toda esta serie: [Complejidad vs. todo lo demás](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md).

## Cuándo pasar de KISS a SOLID

KISS y SOLID no compiten — son dos momentos distintos del mismo código. La pregunta real no es "¿aplico KISS o SOLID?", es **"¿en qué momento dejo de aplicar KISS y empiezo a refactorizar hacia SOLID?"**

La respuesta: **cuando el código simple empieza a doler.** No antes. El código se escribe simple y directo mientras cumple lo que se le pide (KISS + YAGNI). El momento exacto de refactorizar hacia SOLID es cuando un cambio de requerimiento real vuelve ese código simple difícil de manejar — ni un día antes, por las dudas, ni un día después, ya con el dolor instalado.

Esto es literalmente la Regla del Boy Scout de [04-deuda-tecnica.md](./04-deuda-tecnica.md) aplicada al momento de diseño, y el último paso del framework de trade-offs de punto 1: [empezar simple, optimizar después](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md).

La madurez técnica no se mide por usar patrones de diseño desde el día uno — se mide por saber reconocer el momento exacto en que hacen falta.
