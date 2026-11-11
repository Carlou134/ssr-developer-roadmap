# Fase 7: Ecosistema moderno, cloud y DevOps

En esta carpeta aprendes **cómo se empaqueta, se construye y se entrega el software de forma reproducible**: qué es un contenedor y cómo se diferencia de una máquina virtual, cómo se trabaja con Docker y Docker Compose en local, y cómo un pipeline de CI/CD lleva el código desde un commit hasta producción.

-----

## Por qué importa

"En mi máquina funciona" no es una respuesta aceptable en un equipo. Hoy casi cualquier proyecto se ejecuta en contenedores y se despliega con un pipeline automático. Quien no entiende qué hay debajo copia un `Dockerfile` de internet, no sabe por qué la imagen pesa 2 GB y no puede leer por qué falla el pipeline.

Un Junior ejecuta `docker compose up` porque el README lo dice. Un Semi-Senior sabe **qué hay dentro de la imagen, por qué se construye así y qué hace cada paso del pipeline**.

-----

## Antes de empezar

Conviene que ya tengas:

* Manejo básico de terminal y nociones de Linux.
* Nociones de arquitectura de sistemas (monolito vs. microservicios): [Arquitectura de sistemas y estilos arquitectónicos](../11-arquitectura-de-sistemas-y-estilos-arquitectonicos/).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Fundamentos de Docker](01-fundamentos-de-docker.md) | Imagen vs. contenedor, contenedor vs. VM, namespaces y cgroups, CLI/daemon/registry, límites de la portabilidad y diferencia entre Docker, Kubernetes y microservicios | Terminal básica |
| [2. Máquinas virtuales vs. contenedores](02-maquinas-virtuales-vs-contenedores.md) | Hipervisores, reservar vs. consumir recursos, límites y OOM, aislamiento y escape de contenedores, snapshots vs. imágenes, y por qué en producción se usan juntos | Lección 1 |

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
