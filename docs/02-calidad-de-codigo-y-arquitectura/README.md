# Fase 2: Calidad de código y arquitectura

En esta carpeta aprendes a **escribir código que otra persona (o tú dentro de seis meses) pueda leer, cambiar y probar sin miedo**: los principios SOLID y los patrones de diseño, cuándo una abstracción vale la pena y cuándo es sobreingeniería, cómo medir y pagar la deuda técnica, y cómo organizar una aplicación con N-Tier, Clean Architecture, CQRS y arquitectura hexagonal.

-----

## Por qué importa

Escribir código que "funciona" es el piso, no el objetivo. Un Junior entrega una función que pasa la prueba manual una vez. Un Semi-Senior entrega código que otra persona puede leer, modificar y en el que puede confiar sin miedo a romper algo. La diferencia no es estética: es el costo de mantener el sistema en el tiempo.

-----

## Antes de empezar

Conviene que ya tengas:

* C# con clases, interfaces, herencia y `async`/`await`.
* Haber creado una API con ASP.NET Core y Entity Framework Core, aunque sea simple.
* La fase anterior, sobre todo [¿Qué son los trade-offs?](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md): casi todas las decisiones de esta fase son trade-offs.

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. SOLID y patrones de diseño](01-solid-y-patrones.md) | Los 5 principios SOLID; Factory, Strategy, Observer, Repository y Command; por qué evitar el Singleton clásico | Clases e interfaces en C# |
| [2. Arquitectura y límites entre capas](02-arquitectura-y-limites.md) | La regla de dependencia, guard clauses, convenciones de nombres y comentarios | SOLID |
| [3. Estrategia de testing](03-estrategia-de-testing.md) | Unitarias, integración y E2E; qué reemplazar con dobles; cobertura útil frente a inflada | SOLID, límites entre capas |
| [4. Deuda técnica](04-deuda-tecnica.md) | Code smells, Big Ball of Mud, cuantificar en impacto de negocio, Regla del Boy Scout y análisis estático | SOLID, testing |
| [5. Clase abstracta frente a interfaz](05-abstract-vs-interface.md) | "Es un" frente a "puede hacer", `sealed`, sobrecarga frente a sobrescritura | SOLID |
| [6. Inyección de dependencias](06-inyeccion-de-dependencias.md) | Transient, Scoped y Singleton; la dependencia cautiva; el patrón Options con validación al arrancar | SOLID (DIP) |
| [7. DRY, KISS y YAGNI](07-principios-de-simplicidad.md) | Los tres principios de simplicidad, la abstracción incorrecta y cuándo pasar de KISS a SOLID | SOLID, deuda técnica |
| [8. Arquitectura en capas (N-Tier)](08-arquitectura-en-capas-ntier.md) | Qué aporta N-Tier y qué no, DTOs, excepciones de negocio frente a Result, la crítica al repositorio genérico, async de punta a punta | Límites entre capas, DI |
| [9. Clean Architecture](09-clean-architecture.md) | La regla de dependencia, las 4 capas, modelo de dominio rico frente a anémico, cómo se ve en .NET | N-Tier |
| [10. CQRS](10-cqrs.md) | Separar commands de queries, el malentendido de "dos bases de datos", MediatR | Clean Architecture, patrón Command |
| [11. Arquitectura hexagonal](11-arquitectura-hexagonal.md) | Puertos y adaptadores en Kotlin, mapeo en los dos bordes, cableado explícito, diferencias reales con Clean | Clean Architecture, DI |

Material de apoyo (no es una lección):

* [Prompts útiles](12-prompts-utiles.md): plantillas listas para pedir ayuda estructurada en los escenarios de esta fase.

-----

## El mapa completo en una mirada

```text
PRINCIPIOS (cómo escribir cada clase)
  SOLID            → una razón para cambiar, extender sin modificar, depender de abstracciones
  DRY/KISS/YAGNI   → un solo lugar por conocimiento, lo más simple, nada "por si acaso"
  Regla de oro     → abstraer solo por dolor de acoplamiento, cambio confirmado o necesidad de probar

HERRAMIENTAS DEL LENGUAJE
  Interfaz         → "puede hacer", varias por tipo, sin estado
  Clase abstracta  → "es un", una sola, con estado y lógica común
  DI               → Transient < Scoped < Singleton; nunca algo corto dentro de algo largo

CALIDAD EN EL TIEMPO
  Testing          → pirámide: muchas unitarias, menos de integración, pocas E2E
  Cobertura        → piso por capa, no objetivo; mide código ejecutado, no reglas verificadas
  Deuda técnica    → identificar (smells) → cuantificar (impacto) → pagar (Boy Scout)

ARQUITECTURA (cómo organizar la aplicación)
  N-Tier           → capas hacia abajo; la testabilidad la agrega DIP
  Clean/Hexagonal  → todo apunta hacia el dominio; el framework y la base son detalles
  CQRS             → escrituras por el dominio, lecturas directo a DTO; una sola base basta

La pregunta de fondo en todas: ¿qué problema concreto tengo hoy que esto resuelve?
```

-----

## Nota de alcance

Las lecciones 8 a 11 cubren N-Tier, Clean Architecture, CQRS y arquitectura hexagonal a **nivel conceptual con ejemplos de código concretos**: suficiente para entender la regla de dependencia, el vocabulario de cada una y cuándo se justifican. Llevarlo a un proyecto completo de punta a punta, no solo fragmentos, sigue siendo un ejercicio práctico pendiente, separado de estas notas.

La comparación de estos estilos a nivel de **sistema** (monolitos, microservicios, eventos) está en la [fase 11](../11-arquitectura-de-sistemas-y-estilos-arquitectonicos/).

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
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento.
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

-----

## Después de esta carpeta

Ya sabes organizar el código por dentro. El siguiente paso es diseñar cómo se comunica con el exterior: [Diseño de APIs y contratos](../03-diseno-de-apis-y-contratos/).
