# Rendimiento y Profiling de Cliente

## Re-renders innecesarios

React re-renderiza un componente cuando cambia su estado, sus props, o el estado de un componente padre. El problema no es que renderice — es cuando renderiza **sin necesidad real**, por ejemplo porque un componente padre se re-renderiza y arrastra a todos sus hijos aunque ninguno cambió de verdad.

Para detectarlo se usa el **React DevTools Profiler** (extensión del navegador): graba una interacción y muestra un flamegraph de qué componentes se re-renderizaron y cuánto tardó cada uno. Sin esta herramienta, "optimizar rendimiento" es adivinar.

Las herramientas para evitar el re-render innecesario:

- `React.memo`: evita que un componente se re-renderice si sus props no cambiaron (comparación superficial).
- `useMemo`: memoriza el *resultado* de un cálculo costoso entre renders.
- `useCallback`: memoriza la *referencia* de una función entre renders.

**La trampa real:** `useMemo`/`useCallback` no son gratis — tienen su propio costo de memoria y comparación. Si los usas en todos lados "por si acaso", en muchos casos el costo de memorizar es mayor que el de simplemente recalcular. Y hay un error clásico: memorizar un objeto o array literal (`{ id: 1 }`) en cada render igual crea una referencia nueva cada vez, rompiendo la memorización de un `React.memo` hijo que depende de esa prop — por eso hay que memorizar también el objeto que se pasa como prop, no solo el componente. La regla práctica: mide primero con el Profiler, memoriza donde el flamegraph muestra un problema real, no de forma preventiva en todo el árbol.

## Web Vitals

Google mide la "salud" de rendimiento percibido de una página con 3 métricas centrales (Core Web Vitals):

- **LCP (Largest Contentful Paint):** cuánto tarda en pintarse el elemento más grande visible (normalmente la imagen o bloque de texto principal). Objetivo: menor a 2.5s.
- **CLS (Cumulative Layout Shift):** cuánto "salta" el layout mientras carga (ej. una imagen sin dimensiones reservadas empuja el texto hacia abajo cuando termina de cargar). Objetivo: menor a 0.1.
- **INP (Interaction to Next Paint):** cuánto tarda la interfaz en responder visualmente después de que el usuario hace clic, toca o escribe. Objetivo: menor a 200ms.

Nota importante si estudiaste esto con material más antiguo: hasta marzo de 2024, la tercera métrica oficial era **FID** (First Input Delay), que solo medía el retraso del *primer* input. INP la reemplazó porque mide la capacidad de respuesta durante *toda* la sesión, no solo el primer clic — es una métrica más completa y más difícil de "hackear" optimizando solo el primer evento.

## Lazy Loading

Cargar todo el JavaScript de la aplicación de una sola vez infla el bundle inicial. **Code splitting** divide el bundle en pedazos que se cargan solo cuando hacen falta:

- `import()` dinámico + `React.lazy` + `Suspense`: carga un componente solo cuando se necesita renderizar (ej. un modal que casi nunca se abre).
- `next/dynamic` en Next.js: hace lo mismo con soporte adicional para desactivar SSR en un componente específico (`ssr: false`) cuando ese componente depende de APIs del navegador.
- Code splitting por ruta: Next.js ya hace esto automáticamente — cada página es su propio chunk, el usuario no descarga el JavaScript de rutas que no visita.

## Memory Leaks en el cliente

Un memory leak en frontend suele venir de una de estas 3 causas:

1. **Suscripciones sin limpiar:** un `useEffect` que agrega un event listener (`window.addEventListener`) o una suscripción (WebSocket, `setInterval`) sin devolver una función de limpieza que lo remueva al desmontar el componente.
2. **Closures que retienen estado grande:** una función guardada en una variable externa al componente (ej. un callback pasado a una librería externa) que sigue referenciando el `state` de un componente ya desmontado.
3. **`setState` después de desmontar:** una petición asíncrona que responde después de que el componente ya se desmontó e intenta actualizar estado que ya no existe.

Para detectarlos: la pestaña **Memory** de Chrome DevTools permite tomar heap snapshots antes y después de una interacción repetida (ej. abrir y cerrar un modal 10 veces) y comparar cuánta memoria no se liberó. Si el heap crece de forma sostenida sin bajar, hay un leak. Esto es un uso distinto de DevTools al de inspeccionar y editar CSS/HTML en la pestaña Elements — ambas pestañas están en la misma herramienta, pero resuelven problemas completamente distintos.
