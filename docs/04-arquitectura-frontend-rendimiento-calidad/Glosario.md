# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**AbortController.** API del navegador para cancelar una petición `fetch` en curso. Se usa para evitar condiciones de carrera.

**Backoff exponencial.** Esperar cada vez más tiempo entre reintentos: 1 s, 2 s, 4 s, 8 s.

**Bundle.** El JavaScript que el navegador tiene que descargar para la aplicación.

**Caché.** Copia guardada de una respuesta para no volver a pedirla.

**Client Component.** Componente que se ejecuta en el navegador (y también se prerenderiza en el servidor); permite hooks y eventos. Se marca con `"use client"`.

**CLS (*Cumulative Layout Shift*).** Core Web Vital que mide cuánto "salta" el contenido mientras carga. Bueno: ≤ 0,1.

**Code splitting.** Dividir el bundle en partes que se descargan solo cuando hacen falta.

**Condición de carrera.** El resultado depende del orden en que terminan operaciones asíncronas, como dos peticiones que responden en distinto orden.

**Core Web Vitals.** Las tres métricas de Google para la experiencia percibida: LCP, INP y CLS.

**CSR (*Client-Side Rendering*).** El servidor envía un HTML casi vacío y JavaScript arma la página en el navegador.

**Custom hook.** Función que empieza con `use` y reutiliza lógica con estado entre componentes. Comparte lógica, no estado.

**Estado de cliente.** Datos cuyo dueño es el navegador, como un formulario sin enviar o un panel abierto.

**Estado de servidor.** Datos cuyo dueño es el backend; el navegador solo tiene una copia que puede quedar vieja.

**FID (*First Input Delay*).** Métrica que medía el retraso del primer input. INP la reemplazó en marzo de 2024.

**Flamegraph.** Gráfico que muestra qué componentes se renderizaron y cuánto tardó cada uno.

**`gcTime`.** En TanStack Query, cuánto tiempo se guarda en caché un dato que ningún componente usa. Antes de la v5 se llamaba `cacheTime`.

**Handler (de MSW).** La definición de qué responder para una ruta concreta.

**Heap snapshot.** Fotografía de todos los objetos en memoria en un instante.

**Hidratación.** El proceso por el que React conecta el JavaScript al HTML que llegó del servidor, para volverlo interactivo.

**Hidratación limpia.** Que el cliente herede la caché de datos del servidor, sin volver a pedirlos ni mostrar un parpadeo.

**Hydration mismatch.** El HTML del servidor no coincide con lo que React genera en el cliente al hidratar.

**Igualdad referencial.** Dos objetos son "iguales" para React solo si son el mismo objeto en memoria, no si tienen el mismo contenido.

**INP (*Interaction to Next Paint*).** Core Web Vital que mide cuánto tarda la interfaz en responder a una interacción. Bueno: ≤ 200 ms.

**Invalidación.** Marcar una entrada de la caché como obsoleta para que se vuelva a pedir.

**jsdom.** Implementación del DOM en Node, para ejecutar pruebas de UI sin un navegador.

**Lazy loading.** Cargar un recurso (componente, imagen) recién cuando se necesita.

**LCP (*Largest Contentful Paint*).** Core Web Vital que mide cuánto tarda en pintarse el elemento más grande visible. Bueno: ≤ 2,5 s.

**Memoizar.** Guardar el resultado de un cálculo o una referencia para no recalcularla si sus entradas no cambiaron.

**Memory leak (fuga de memoria).** Memoria que la aplicación ya no usa pero que no se libera.

**MPA (*Multi-Page Application*).** Cada navegación pide al servidor una página HTML completa.

**MSW (*Mock Service Worker*).** Librería que intercepta peticiones HTTP a nivel de red y responde con datos simulados.

**Prop drilling.** Pasar una prop por varios componentes intermedios que no la usan, solo para que llegue a uno que sí.

**Prueba frágil.** Prueba que se rompe con un refactor aunque el comportamiento no haya cambiado.

**Query (de RTL).** Función para buscar elementos en la pantalla, como `getByRole`, `queryByText` o `findByLabelText`.

**Query key.** El identificador de una consulta en la caché de TanStack Query, como `["pedidos", { estado }]`.

**React Compiler.** Herramienta de build que agrega la memoización automáticamente. Versión estable 1.0 en octubre de 2025.

**React Testing Library (RTL).** Librería para probar componentes de React desde el punto de vista del usuario.

**Render (re-render).** Cuando React vuelve a ejecutar un componente para calcular qué mostrar.

**Rol accesible (ARIA role).** Lo que un elemento es para las tecnologías de asistencia: botón, campo de texto, encabezado.

**Serializable.** Un valor que se puede convertir a texto para enviarlo por la red, como un string, un número o un objeto plano.

**Server Action.** Función marcada con `"use server"` que se ejecuta en el servidor cuando la llama un formulario o un Client Component.

**Server Component.** Componente de React que se ejecuta solo en el servidor y no envía su código al navegador.

**SPA (*Single-Page Application*).** El HTML se carga una vez y JavaScript actualiza la pantalla pidiendo solo datos.

**SSR (*Server-Side Rendering*).** Generar el HTML en el servidor, con los datos ya incluidos, en el momento de la petición. No confundir con "SSR" de Semi-Senior.

**Stale (obsoleto).** Un dato en caché que se considera viejo y conviene revalidar.

**Stale-while-revalidate.** Mostrar el dato en caché de inmediato mientras se pide uno fresco en segundo plano.

**`staleTime`.** En TanStack Query, cuánto tiempo un dato se considera fresco y no se revalida. Por defecto, `0`.

**Test runner.** Herramienta que encuentra, ejecuta y reporta las pruebas, como Jest o Vitest.

**`"use client"`.** Directiva que marca el límite a partir del cual los componentes son de cliente. Todo lo que importa ese archivo también pasa al cliente.

**`user-event`.** Librería que simula interacciones reales del usuario: escribir, hacer clic, tabular.

**Virtualización.** Renderizar solo los elementos visibles de una lista larga e ir reemplazándolos al hacer scroll.
