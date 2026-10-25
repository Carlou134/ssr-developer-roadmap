# Fase 1: Autonomía y Resolución de Problemas

En esta carpeta aprendes a **resolver un bloqueo sin depender de que alguien te salve**: a investigar antes de escalar, a usar Git y el depurador para encontrar el origen de un bug, a leer logs y stack traces sin perder el rastro, y a evaluar alternativas con criterio cuando no hay una única respuesta correcta.

-----

## Por qué importa

La diferencia entre un Junior y un Semi-Senior no es cuánto código escriben, es **cómo enfrentan un bloqueo**. Un Junior se traba y espera que alguien le resuelva el problema. Un SSR investiga, aísla, prueba hipótesis y solo escala con contexto, no con un "no funciona, ayuda".

-----

## Antes de empezar

Conviene que ya tengas:

* Lo básico de Git: commits, ramas y `git log`.
* Haber ejecutado una aplicación .NET (o de otro stack) desde un IDE.
* Haber visto alguna vez una excepción en consola, aunque no la hayas entendido del todo.

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Investigación y criterio](01-investigacion-y-criterio.md) | Investigar antes de escalar, `git bisect`, `git blame`, `git log -p`, MRE y timeboxing | Git básico |
| [2. Depuración avanzada](02-depuracion-avanzada.md) | Breakpoints condicionales (IntelliJ, VS 2022, VS Code), step over/into/out, watch, call stack y profiling | Investigación y criterio |
| [3. Interpretación de logs](03-interpretacion-de-logs.md) | Logging estructurado, niveles de log, correlation ID, `ILogger` y configuración en `appsettings.json` | Depuración avanzada |
| [4. Stack traces](04-stack-traces.md) | Leer una excepción, `try`/`catch` anidados, modelo de dos pasadas del CLR y `throw;` frente a `throw ex;` | Depuración avanzada, interpretación de logs |
| [5. ¿Qué son los trade-offs?](05-que-son-los-tradeoffs.md) | Qué es un trade-off, los 8 principales y un framework práctico para decidir | Investigación y criterio |

Material de apoyo (no es una lección):

* [Prompts útiles](06-prompts-utiles.md): plantillas listas para pedir ayuda estructurada en los escenarios de esta fase.

-----

## El mapa completo en una mirada

```text
Bloqueo
  │
  ├─ 1. Investigar      → 2-3 alternativas antes de escalar, timebox definido
  ├─ 2. Reproducir      → MRE: el caso mínimo que falla siempre
  ├─ 3. Ubicar el cambio → git bisect (qué commit), git blame / git log -p (por qué)
  ├─ 4. Observar        → depurador: breakpoints condicionales, watch, call stack
  ├─ 5. Leer evidencia  → logs estructurados + correlation ID, stack trace de arriba hacia abajo
  └─ 6. Decidir         → evaluar trade-offs, no buscar la solución "perfecta"

¿Se agotó el timebox?  → escalar CON contexto: qué probaste, qué viste, qué sospechas
```

-----

## Nota importante: sintaxis frente a comportamiento

Cuando compares estos conceptos con otro lenguaje o stack (Java, Python, Node), no asumas que solo cambia la sintaxis. Antes de aplicar una regla aprendida en .NET a otro entorno, verifica si también cambió el **comportamiento** de fondo.

* **Cambia solo el nombre (sintaxis, sin sorpresas):** `LogLevel.Critical` en `ILogger` frente a `Fatal` en Serilog. Es el mismo nivel de severidad con distinto nombre.
* **Cambia el comportamiento real (no es cosmético):** en .NET, `throw ex;` dentro de un `catch` resetea el stack trace porque el CLR lo captura en el momento del `throw`. Java y JavaScript capturan el trace al **crear** la excepción, no al lanzarla, así que esa regla específica de .NET no aplica igual ahí.

Regla práctica: el concepto lo aprendes una vez y aplica en todos lados. El comportamiento exacto lo verificas cada vez que saltas de stack; no lo asumes por analogía.

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura, así sabes dónde buscar cada cosa:

1. **En una frase:** qué es, sin jerga.
2. **Antes de empezar:** qué tienes que saber y las palabras nuevas.
3. **El problema:** una situación concreta que muestra por qué existe la herramienta.
4. **Cómo funciona:** el paso a paso, con ejemplos cortos y explicados.
5. **Ejemplo completo:** todo junto, aplicado al mismo problema.
6. **Errores comunes:** lo que suele salir mal, por qué pasa y cómo se arregla.
7. **Evolución y versiones:** qué cambió con el tiempo (solo si aplica).
8. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
9. **Resumen en 5 líneas:** para repasar.
10. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento (solo si aplica).
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

-----

## Después de esta carpeta

Ya sabes resolver problemas con autonomía. El siguiente paso es escribir código que genere menos problemas: [Calidad de código y arquitectura](../02-calidad-de-codigo-y-arquitectura/).
