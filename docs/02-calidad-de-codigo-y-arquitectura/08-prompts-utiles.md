# Prompts Útiles: Calidad de Código y Arquitectura

Prompts base para pedir ayuda de forma estructurada en los escenarios de esta fase — copiar, completar los corchetes, usar.

## Revisión completa (SOLID + code smells)

> Revisa este código de C# buscando específicamente: violaciones a los principios SOLID (nombra cuál principio se rompe y por qué), code smells clásicos (Clase Dios, Switch Statements, Refused Bequest, Duplicate Code, Speculative Generality), y abstracciones no justificadas (¿resuelven un problema real hoy, o son "por si acaso"?). Para cada hallazgo, explícame el problema concreto que puede causar, no solo que "está mal". Prioriza lo más importante primero.
>
> [PEGAR CÓDIGO ACÁ]

## ¿Está justificada esta abstracción?

> Estoy por crear [interfaz/patrón] para resolver [problema]. Según la Regla de Oro (dolor de acoplamiento ya presente, cambio frecuente ya confirmado, o necesidad real de aislar para testing), ¿está justificada, o es sobre-ingeniería/YAGNI? Explícame con un ejemplo de mi caso concreto.

## Revisión de naming

> Revisa los nombres de clases, interfaces y métodos en este código. Para cada uno que no describa con precisión su responsabilidad real (ES-UN vs. PUEDE-HACER en interfaces), dime un nombre más preciso y por qué el actual es engañoso.
>
> [PEGAR CÓDIGO ACÁ]

## ¿Es testeable?

> Revisa esta clase y dime si es fácil de testear en aislamiento, o si tiene dependencias difíciles de mockear. Si hay un problema, dime qué principio SOLID lo está bloqueando (probablemente DIP) y cómo refactorizarlo.
>
> [PEGAR CÓDIGO ACÁ]

## Deuda técnica con impacto real

> Mira este módulo y dime qué deuda técnica identificas. Para cada una, ayúdame a redactarla como un ticket — no "esto está feo", sino el impacto real en tiempo/riesgo si no se arregla.
>
> [PEGAR CÓDIGO ACÁ]
