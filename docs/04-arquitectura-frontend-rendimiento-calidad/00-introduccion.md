# Arquitectura Frontend, Rendimiento y Calidad

## ¿Por qué este punto?

Escribir un componente que "funciona" es fácil. La diferencia entre un desarrollador Junior y uno SSR en frontend está en decisiones que no se ven a simple vista: dónde vive el estado, qué corre en el servidor y qué corre en el navegador, por qué la app se siente lenta aunque el código "está bien", y cómo se prueba una interfaz de forma que la prueba realmente proteja contra regresiones.

Este punto se apoya en React/Next.js por ser el stack que mencionas usar, pero los conceptos (estado de servidor vs. cliente, hidratación, rendimiento percibido, testing de comportamiento) aplican igual en Vue, Angular o Svelte.

## Contenido

1. [Estado de Cliente vs. Servidor](./01-estado-cliente-vs-servidor.md) — por qué no son lo mismo, TanStack Query/SWR, caché, invalidación, reintentos automáticos.
2. [Server/Client Components](./02-server-client-components.md) — de SPA a arquitecturas híbridas, hidratación, Custom Hooks, Prop Drilling.
3. [Rendimiento y Profiling](./03-rendimiento-y-profiling.md) — re-renders, Web Vitals (LCP, CLS, INP), Lazy Loading, memory leaks.
4. [Estrategia de Testing en UI](./04-estrategia-de-testing-ui.md) — Jest/Vitest, React Testing Library, MSW, estados de carga/error.
5. [Prompts Útiles](./05-prompts-utiles.md) — plantillas para debugging de rendimiento, diseño de hooks y estrategia de testing.
