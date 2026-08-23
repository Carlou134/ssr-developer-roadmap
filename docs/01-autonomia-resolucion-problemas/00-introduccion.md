# Fase 1: Autonomía y Resolución de Problemas

## ¿Por qué importa?

La diferencia entre un Junior y un Semi-Senior no es cuánto código escriben, es CÓMO enfrentan un bloqueo. Un Junior se traba y espera que alguien le resuelva el problema. Un SSR investiga, aísla, prueba hipótesis, y solo escala con contexto — no con un "no funciona, ayuda".

Esta fase cubre las herramientas y el criterio para resolver problemas sin depender de que alguien te salve.

## Contenido

1. [Investigación y criterio](./01-investigacion-y-criterio.md) — cómo investigar antes de escalar, Git como herramienta de debugging, MRE, timeboxing.
2. [Depuración avanzada](./02-depuracion-avanzada.md) — breakpoints condicionales (IntelliJ y VS 2022), step over/into/out, watch, call stack, profiling.
3. [Interpretación de logs](./03-interpretacion-de-logs.md) — logging estructurado, niveles de log, correlation ID.
4. [Stack Traces](./04-stack-traces.md) — cómo leer una excepción sin perder el rastro real del error.
5. [¿Qué son los Trade-offs?](./05-que-son-los-tradeoffs.md) — el vocabulario base para evaluar alternativas: qué es un trade-off, los 8 principales, y un framework práctico para navegarlos.

## Nota importante: sintaxis vs comportamiento

Cuando compares estos conceptos con otro lenguaje o stack (Java, Python, Node), no asumas que solo cambia la sintaxis. Antes de aplicar una regla aprendida en .NET a otro entorno, verifica si también cambió el **comportamiento** de fondo.

- **Cambia solo el nombre (sintaxis, sin sorpresas):** `LogLevel.Critical` en `ILogger` vs `Fatal` en Serilog — mismo nivel de severidad, distinto nombre.
- **Cambia el comportamiento real (no es cosmético):** en .NET, `throw ex;` dentro de un catch resetea el stack trace porque el CLR lo captura en el momento del `throw`. Java y JavaScript capturan el trace al **crear** la excepción, no al lanzarla — esa regla específica de .NET no aplica igual ahí.

Regla práctica: el concepto lo aprendes una vez y aplica en todos lados. El comportamiento exacto, lo verificas cada vez que saltas de stack — no lo asumes por analogía.
