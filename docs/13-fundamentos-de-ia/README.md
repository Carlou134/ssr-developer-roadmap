# Fase 13: Fundamentos de IA para desarrolladores

En esta carpeta aprendes **cómo trabaja un modelo de lenguaje desde el punto de vista de quien lo integra**: qué es un token y por qué define el costo y los límites, cómo se conecta un modelo con sistemas externos mediante MCP y cómo se le enseñan procedimientos reutilizables con Skills.

-----

## Por qué importa

Hoy casi cualquier equipo usa un asistente de código o integra un LLM en su producto. La diferencia entre usarlo y **entenderlo** se nota rápido: quien no sabe qué es un token no entiende por qué la factura se dispara ni por qué el modelo "se olvida" de lo que le dijiste al principio; quien no distingue MCP de Skills conecta todo a todo y termina con un agente lento, caro e inseguro.

Un Junior usa la herramienta. Un Semi-Senior sabe **qué entra al contexto, cuánto cuesta y qué puede hacer el modelo con lo que le das**.

-----

## Antes de empezar

Conviene que ya tengas:

* Nociones de APIs HTTP y JSON: [Estándares RESTful](../03-diseno-de-apis-y-contratos/01-estandares-restful.md).
* Ideas básicas de seguridad (mínimo privilegio, inyección): [Fundamentos de seguridad](../05-seguridad-y-resiliencia/01-fundamentos-de-seguridad.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Tokens y ventana de contexto](01-tokens-y-ventana-de-contexto.md) | Qué es un token, ventana de contexto, tokens de entrada y salida, costo, `max_tokens` y por qué la conversación "se olvida" | APIs HTTP |
| [2. MCP (Model Context Protocol)](02-mcp-model-context-protocol.md) | Tool use, el problema N×M, host/cliente/servidor, tools, resources y prompts, transportes y seguridad | Lección 1, JSON |
| [3. Skills](03-skills.md) | Qué es una skill, `SKILL.md`, carga progresiva y cómo se diferencia de MCP, del prompt de sistema y de `CLAUDE.md` | Lecciones 1 y 2 |

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
