# Fase 12: Estructuras de datos y algoritmos

En esta carpeta aprendes a **organizar los datos para que tu código siga siendo rápido cuando el volumen crece**: qué es una estructura de datos, cómo medir el costo de una operación con la notación asintótica, las estructuras clásicas y los algoritmos que se apoyan en ellas.

-----

## Por qué importa

La mayoría de los problemas de rendimiento de un sistema real no vienen de la falta de servidores, sino de una mala elección de estructura: una búsqueda lineal dentro de un bucle, una lista donde hacía falta un diccionario. Funcionan con los datos de desarrollo y explotan con los de producción.

Un Junior hace que funcione con 100 registros. Un Semi-Senior sabe **cuánto va a tardar con un millón** antes de ejecutarlo.

-----

## Antes de empezar

Conviene que ya tengas:

* C# básico: tipos, clases, `List<T>`, `Dictionary<TKey, TValue>` y LINQ.
* Testing básico, para validar tus soluciones: [Estrategia de testing](../02-calidad-de-codigo-y-arquitectura/03-estrategia-de-testing.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Por qué estructuras de datos](01-por-que-estructuras-de-datos.md) | Datos y algoritmos, las cuatro funciones de una estructura, acceso por índice y por clave, memoria estática y dinámica, cómo elegir | Colecciones básicas de C# |
| [2. API de una estructura de datos](02-api-de-una-estructura-de-datos.md) | API pública frente a implementación, encapsulación en C# y JS (`#`), APIs restringidas, fugas de referencias e interfaces como contrato | Lección 1, interfaces |

-----

## Práctica fuera de esta carpeta

Aquí están los conceptos. Los ejercicios con código y tests viven en el repositorio [dsa-and-algorithms](https://github.com/Carlou134).

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura, así sabes dónde buscar cada cosa:

1. **En una frase:** qué es, sin jerga.
2. **Antes de empezar:** qué tienes que saber y las palabras nuevas.
3. **El problema:** una situación concreta que muestra por qué existe el concepto.
4. **Cómo funciona:** el paso a paso, con ejemplos cortos y explicados.
5. **Ejemplo completo:** todo junto, aplicado al mismo problema.
6. **Errores comunes:** lo que suele salir mal, por qué pasa y cómo se arregla.
7. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
8. **Resumen en 5 líneas:** para repasar.
9. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
10. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento.
11. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
12. **Siguiente lección:** hacia dónde continuar.
