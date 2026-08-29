# Prompts Útiles: Autonomía y Resolución de Problemas

Prompts base para pedir ayuda de forma estructurada en los escenarios de esta fase — copiar, completar los corchetes, usar.

## Investigación antes de escalar

> Tengo este problema: [descripción]. Ya probé: [alternativa 1, resultado], [alternativa 2, resultado]. Mi hipótesis es [hipótesis]. Ayúdame a evaluar si mi diagnóstico tiene sentido, o si me estoy perdiendo algo, antes de escalarlo.

## Lectura de un stack trace

> Tengo este stack trace de .NET. Ayúdame a identificar el frame más profundo que sea código mío (no del framework), qué tipo de excepción es y qué la causa típicamente, y si el mensaje alcanza para diagnosticar o necesito más contexto.
>
> [PEGAR STACK TRACE ACÁ]

## Reconstruir una secuencia de logs

> Tengo estos logs de una transacción, mismo Correlation ID, en orden de timestamp. Ayúdame a reconstruir qué pasó y en qué paso está probablemente el problema real.
>
> [PEGAR LOGS ACÁ]

## `git bisect` guiado

> Necesito encontrar en qué commit se rompió [funcionalidad]. En el commit [hash/tag] andaba bien. Guíame paso a paso con `git bisect`, y ayúdame a decidir en cada parada si marco good o bad según lo que te describa.

## ¿Sigo o escalo? (chequeo de timeboxing)

> Llevo [tiempo] investigando [problema]. Probé: [...]. ¿Ya tengo contexto suficiente para escalar con criterio, o vale la pena seguir un poco más por [pista puntual]?
