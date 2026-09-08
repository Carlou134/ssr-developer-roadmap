# Prompts Útiles: Arquitectura Frontend, Rendimiento y Calidad

## Debuggear re-renders innecesarios

> Tengo este componente que se re-renderiza más de lo esperado: [pegar código]. Ayúdame a identificar qué prop o estado está causando el re-render innecesario, si `React.memo`/`useMemo`/`useCallback` resolvería el problema de forma justificada (no preventiva), y si hay algún objeto o función literal creado en cada render que esté rompiendo la memorización de un hijo.

## Decidir si algo es estado de cliente o de servidor

> Tengo este dato en mi aplicación: [describir el dato, de dónde viene, quién lo puede cambiar]. Ayúdame a decidir si debería vivir en estado de cliente (Context/Zustand) o gestionarse con TanStack Query/SWR como estado de servidor, y por qué.

## Diseñar un Custom Hook

> Tengo esta lógica repetida en varios componentes: [pegar la lógica duplicada]. Ayúdame a extraerla en un Custom Hook reutilizable, definiendo qué recibe como parámetros y qué debería devolver.

## Revisar estrategia de testing de un componente

> Tengo este componente que consume una API: [pegar código]. Ayúdame a diseñar los tests con React Testing Library y MSW cubriendo los 4 estados: carga, éxito con datos, éxito vacío, y error — no solo el caso feliz.
