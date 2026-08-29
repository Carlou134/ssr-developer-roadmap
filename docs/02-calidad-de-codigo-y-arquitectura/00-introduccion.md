# Fase 2: Calidad de Código y Arquitectura

## ¿Por qué importa?

Escribir código que "funciona" es el piso, no el objetivo. Un Junior entrega una función que pasa la prueba manual una vez. Un Semi-Senior entrega código que otra persona (o él mismo, seis meses después) puede leer, modificar y confiar sin miedo a romper algo. La diferencia no es estética — es directamente el costo de mantener el sistema en el tiempo.

Esta fase cubre los principios, patrones y prácticas que sostienen esa diferencia.

## Contenido

1. [SOLID y Patrones de Diseño](./01-solid-y-patrones.md) — los 5 principios SOLID, y cuándo (y cuándo NO) usar Factory, Strategy, Observer y Repository.
2. [Arquitectura y Límites entre Capas](./02-arquitectura-y-limites.md) — por qué el dominio no debe conocer la base de datos, Guard Clauses, convenciones de naming.
3. [Estrategia de Testing](./03-estrategia-de-testing.md) — Unit, Integration y E2E: qué prueba cada una, cuándo corre cada una, y qué es una cobertura útil.
4. [Deuda Técnica](./04-deuda-tecnica.md) — cómo identificarla, cuantificarla, y la Regla del Boy Scout, más herramientas de análisis estático.
5. [Clase Abstracta vs. Interface](./05-abstract-vs-interface.md) — ES-UN vs. PUEDE-HACER, `sealed` vs `abstract`, sobrecarga vs sobrescritura.
6. [Inyección de Dependencias](./06-inyeccion-de-dependencias.md) — los 3 lifetimes (Transient/Scoped/Singleton) y el Patrón Options.
7. [DRY, KISS y YAGNI](./07-principios-de-simplicidad.md) — los tres principios de simplicidad, cruzados con los trade-offs de punto 1.
8. [Prompts Útiles](./08-prompts-utiles.md) — plantillas listas para pedir ayuda estructurada en los escenarios de esta fase.

## Nota de alcance

Esta fase cubre arquitectura a **nivel de principio y disciplina** — por qué importa respetar límites entre capas, no un desarrollo completo de una arquitectura específica (Clean, Onion, Hexagonal). Eso último merece su propio espacio práctico, construyéndolo en código real, no solo como notas.
