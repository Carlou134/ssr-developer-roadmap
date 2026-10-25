# Fase 4: Arquitectura frontend, rendimiento y calidad

En esta carpeta aprendes las decisiones de frontend que no se ven a simple vista: **dónde vive cada tipo de estado, qué se ejecuta en el servidor y qué en el navegador, por qué una aplicación se siente lenta aunque el código "esté bien", y cómo probar una interfaz** de forma que la prueba proteja contra regresiones reales.

-----

## Por qué importa

Escribir un componente que "funciona" es fácil. La diferencia entre un Junior y un Semi-Senior en frontend está en las decisiones que no se ven: un store global con datos que quedan viejos, un `"use client"` en la raíz que convierte todo en una SPA de 500 KB, un `React.memo` que no evita ningún render, o pruebas que se rompen con cada refactor y pasan cuando la API se cae.

Esta fase usa React y Next.js, pero los conceptos (estado de servidor frente a cliente, hidratación, rendimiento percibido, pruebas de comportamiento) aplican igual en Vue, Angular o Svelte.

-----

## Antes de empezar

Conviene que ya tengas:

* React con componentes, props, `useState` y `useEffect`.
* TypeScript básico.
* Cómo responde una API REST: [Estándares RESTful](../03-diseno-de-apis-y-contratos/01-estandares-restful.md).
* La pirámide de testing: [Estrategia de testing](../02-calidad-de-codigo-y-arquitectura/03-estrategia-de-testing.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Estado de cliente frente a estado de servidor](01-estado-cliente-vs-servidor.md) | Por qué no son lo mismo; TanStack Query y SWR: caché, query keys, invalidación, reintentos, `staleTime`; hidratación limpia con SSR | React básico, APIs REST |
| [2. Server y Client Components](02-server-client-components.md) | De la SPA al modelo híbrido, el límite de `"use client"`, props serializables, hydration mismatch, custom hooks y prop drilling | Estado de cliente y servidor |
| [3. Rendimiento y profiling](03-rendimiento-y-profiling.md) | Renders innecesarios e igualdad referencial, React Compiler, Core Web Vitals (LCP, INP, CLS), code splitting y memory leaks | Server y Client Components |
| [4. Estrategia de testing en UI](04-estrategia-de-testing-ui.md) | Jest frente a Vitest, React Testing Library, prioridad de queries, `user-event`, MSW y los cuatro estados de un componente | Estado de servidor, testing de la fase 2 |

Material de apoyo (no es una lección):

* [Prompts útiles](05-prompts-utiles.md): plantillas para depurar rendimiento, diseñar hooks y armar la estrategia de testing.

-----

## El mapa completo en una mirada

```text
¿DÓNDE VIVE EL ESTADO?
  ¿Hay que hacer una petición para obtenerlo?
    Sí → estado de servidor → TanStack Query / SWR (caché, key, invalidación, reintentos)
    No → estado de cliente  → useState · Context · Zustand

¿DÓNDE SE EJECUTA EL COMPONENTE?
  ¿Necesita estado, eventos o APIs del navegador?
    No → Server Component  → sin JavaScript al navegador, datos directos
    Sí → Client Component  → "use client" en la hoja interactiva, no en la raíz
  Props que cruzan al cliente: serializables · render determinista para evitar mismatch

¿POR QUÉ ES LENTO?  (medir primero)
  Renders    → Profiler de React → bajar el estado · referencias estables · React Compiler
  Percepción → LCP ≤ 2,5 s · INP ≤ 200 ms · CLS ≤ 0,1 (percentil 75, datos de campo)
  Descarga   → code splitting por ruta · lazy para lo pesado (nunca la imagen principal)
  Memoria    → heap snapshots · todo useEffect que suscribe, limpia

¿CÓMO SE PRUEBA?
  Como el usuario: getByRole → user-event → findBy · red simulada con MSW
  Cuatro estados: cargando · éxito · vacío · error
```

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura, así sabes dónde buscar cada cosa:

1. **En una frase:** qué es, sin jerga.
2. **Antes de empezar:** qué tienes que saber y las palabras nuevas.
3. **El problema:** una situación concreta que muestra por qué existe la herramienta.
4. **Cómo funciona:** el paso a paso, con ejemplos cortos y explicados.
5. **Ejemplo completo:** todo junto, aplicado al mismo problema.
6. **Errores comunes:** lo que suele salir mal, por qué pasa y cómo se arregla.
7. **Evolución y versiones:** qué cambió con el tiempo, para reconocer código y tutoriales viejos.
8. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
9. **Resumen en 5 líneas:** para repasar.
10. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento.
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

-----

## Después de esta carpeta

Con esto cierras la arquitectura de aplicación de punta a punta: backend (fase 2), contrato (fase 3) y frontend (fase 4). El siguiente punto del [roadmap](../../README.md) es seguridad y resiliencia.
