# Server/Client Components: de la SPA a la arquitectura híbrida

## Contexto: por qué existe este problema

Antes de las SPA (Single-Page Applications), cada clic en un sitio pedía una página HTML completa nueva al servidor (MPA — Multi-Page Application). Las SPA (React, Vue, Angular) resolvieron esto: el HTML inicial se carga una sola vez, y JavaScript en el navegador se encarga de actualizar solo la parte de la pantalla que cambia, pidiendo al servidor únicamente los datos (JSON) que hacen falta.

Esto mejoró mucho la fluidez, pero introdujo dos problemas reales que toda SPA pura arrastra:

- **SEO y first paint:** si el HTML inicial llega casi vacío y todo el contenido lo pinta JavaScript después, el usuario ve una pantalla en blanco unos instantes, y los crawlers que no ejecutan JavaScript bien (o lo ejecutan tarde, en una segunda pasada) tienen más dificultad para indexar el contenido. Es una desventaja real, aunque conviene matizarla: Googlebot moderno sí ejecuta JavaScript, pero lo hace en una segunda oleada de rastreo, lo cual retrasa la indexación — no es que "no vea nada", pero sí es más lento e inseguro que recibir HTML ya armado.
- **Carga inicial pesada:** todo el JavaScript de la aplicación se descarga antes de poder interactuar con casi nada.

(Nota: el problema del botón "Atrás" del navegador que rompía en SPAs antiguas ya está resuelto en la práctica — React Router, Vue Router, etc. manejan el historial del navegador con la History API. No es una desventaja vigente de las SPA modernas.)

## La respuesta: Server Components (Next.js App Router)

Next.js (desde la App Router) no es "volver a las MPA", es un modelo híbrido: cada componente decide si se ejecuta en el servidor o en el cliente.

- **Server Component (por defecto):** se ejecuta solo en el servidor. Puede acceder directamente a una base de datos o a un servicio interno sin exponer una API pública, y **no envía JavaScript al navegador** para ese componente — el HTML ya llega listo. Ideal para todo lo que no necesita interactividad: layouts, listas de datos, contenido estático.
- **Client Component (`"use client"`):** se hidrata en el navegador y sí puede usar `useState`, `useEffect`, manejadores de eventos, y cualquier hook. Necesario para todo lo interactivo: formularios, dropdowns, cualquier cosa que reaccione a un clic.

La regla práctica: empieza asumiendo que un componente es de servidor, y solo lo marcas como cliente cuando de verdad necesita interactividad o hooks de React. Es habitual ver Server Components "envolviendo" Client Components pequeños y específicos, no al revés.

## Hidratación: dónde puede romperse

"Hidratar" es el proceso donde React toma el HTML que ya llegó del servidor y le "conecta" el JavaScript en el cliente para que se vuelva interactivo, sin volver a pintar todo desde cero. Un **hydration mismatch** ocurre cuando el HTML que generó el servidor no coincide exactamente con lo que React renderizaría en el cliente — causas típicas: usar `Date.now()` o `Math.random()` directamente en el render, o leer `window`/`localStorage` fuera de un `useEffect`. React lo detecta y lanza un warning (o error) en consola, y en el peor caso re-renderiza todo desde el cliente, perdiendo la ventaja del SSR.

(Esto se conecta con la "hidratación limpia" de datos que viste en el archivo anterior — son dos capas distintas del mismo concepto: una es sobre el *HTML* que coincide entre servidor y cliente, la otra es sobre los *datos* que no se vuelven a pedir de más.)

## Custom Hooks: extraer lógica sin duplicarla

Un Custom Hook (`useAlgo`) es una función que empieza con `use` y permite reutilizar lógica con estado entre varios componentes — por ejemplo, `useDebounce`, `useLocalStorage`, o `useFormValidation`. La ventaja frente a copiar y pegar la misma lógica en cada componente es la misma razón detrás de DRY (punto 2): un solo lugar para corregir un bug de esa lógica, no N lugares.

## Prop Drilling: el síntoma y las dos soluciones reales

**Prop Drilling** es pasar una prop a través de 4 o 5 componentes intermedios que no la usan, solo para que llegue al componente final que sí la necesita. Es un code smell de acoplamiento estructural, no solo "molesto de escribir".

Dos soluciones, no intercambiables:

- **Composición:** en vez de pasar datos hacia abajo, pasas los componentes ya armados como `children` o props — el componente intermedio deja de necesitar saber nada del dato.
- **Context API:** cuando el dato realmente es "global" para una sección del árbol (tema, usuario autenticado, idioma), Context evita el drilling exponiendo el valor directamente a quien lo necesite. Ojo: usar Context para todo termina siendo tan problemático como Prop Drilling, porque cualquier cambio en el valor de contexto re-renderiza a todos los consumidores — no es un reemplazo universal de un buen manejo de estado.
