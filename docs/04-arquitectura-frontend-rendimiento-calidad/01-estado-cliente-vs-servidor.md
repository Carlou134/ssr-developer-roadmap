# Estado de Cliente vs. Estado de Servidor

## El error más común: tratarlos como lo mismo

Muchos proyectos meten todo en un único store (Redux, Zustand, Context) — el usuario logueado, el tema oscuro/claro, y también la lista de pedidos que viene de la API. Esto es un error de arquitectura, porque son dos tipos de estado con necesidades completamente distintas:

| | Estado de Cliente | Estado de Servidor |
|---|---|---|
| **Dueño de la verdad** | El navegador (tú decides el valor). | El backend (el navegador solo tiene una copia). |
| **Ejemplos** | Tema oscuro/claro, un modal abierto, un formulario sin enviar. | Lista de productos, perfil del usuario, resultado de una búsqueda. |
| **¿Puede quedar desactualizado?** | No, tú lo controlas. | Sí — alguien más puede cambiar el dato en el servidor. |
| **Herramienta típica** | `useState`, Context, Zustand, Redux. | TanStack Query, SWR. |

El estado de servidor necesita cosas que un store de estado normal no resuelve solo: cachear la respuesta, saber cuándo está "vieja" (stale), reintentar si la petición falla, evitar pedir lo mismo dos veces si dos componentes lo necesitan a la vez, y sincronizarse en segundo plano. Para eso existen librerías como **TanStack Query** (antes React Query) y **SWR**.

## Cómo resuelven esto TanStack Query / SWR

Ambas se basan en la misma idea de HTTP caching: **stale-while-revalidate** — mostrar el dato en caché de inmediato (aunque esté "viejo") mientras se revalida en segundo plano, en vez de mostrar una pantalla de carga cada vez.

- **Caché por clave:** cada consulta se identifica por una key (ej. `["pedidos", userId]`). Si dos componentes piden la misma key, la librería deduplica la petición — solo se llama una vez a la API.
- **Invalidación:** después de una mutación (crear, actualizar, borrar un pedido), le dices a la librería "esta key ya no es válida" y ella vuelve a pedir el dato fresco automáticamente, en vez de que tú actualices el estado local a mano.
- **Reintentos automáticos:** si la petición falla (ej. un timeout de red), reintenta con **backoff exponencial** antes de mostrar un error al usuario — importante para redes móviles inestables.

  **¿Qué es backoff?** Es la estrategia de esperar cada vez más tiempo entre un reintento y el siguiente, en vez de reintentar de inmediato una y otra vez. "Exponencial" significa que ese tiempo de espera se duplica (o crece exponencialmente) en cada intento fallido: por ejemplo, 1s → 2s → 4s → 8s. Sin esto, un cliente que reintenta sin pausa contra un servidor caído o sobrecargado solo empeora el problema (llega a saturarlo más), y en apps móviles agota batería y datos sin necesidad. TanStack Query y SWR traen esta lógica de backoff incorporada por defecto, sin que tengas que implementarla a mano.
- **Refetch en eventos:** por defecto, revalidan cuando la ventana recupera el foco o se recupera la conexión a internet, así el usuario nunca ve datos obsoletos por mucho tiempo sin darse cuenta.

## Hidratación limpia

**¿Qué es SSR?** Server-Side Rendering: el HTML de la página se genera en el servidor en el momento en que el usuario la pide, con los datos ya incluidos — a diferencia de CSR (Client-Side Rendering), donde el servidor manda un HTML casi vacío y JavaScript arma todo después en el navegador.

*Nota para no confundir siglas dentro de este mismo roadmap:* el "SSR" del título de este repositorio (Semi-Senior) es una sigla completamente distinta a la de Server-Side Rendering. Coinciden en las letras, no en el significado.

En una app con SSR (Next.js), el servidor ya hizo la petición de datos para renderizar el HTML inicial. El problema: si el cliente vuelve a pedir esos mismos datos apenas carga la página, hay un parpadeo (loading → datos) que no debería existir, porque el servidor ya los tenía.

La solución es pasarle a TanStack Query/SWR ese dato inicial como `initialData` (o usar el patrón `dehydrate`/`hydrate` de TanStack Query) para que el cliente "herede" la caché del servidor en vez de arrancar en blanco. Esto es lo que se conoce como hidratación limpia: el cliente retoma exactamente donde el servidor dejó el estado, sin refetch innecesario ni parpadeo visual.

## Conexión con el resto del roadmap

Esta distinción (estado de cliente vs. servidor) es el mismo principio de "separar responsabilidades" que viste en SOLID (punto 2) aplicado a datos en vez de a clases: mezclar ambos tipos de estado en un solo store es, en la práctica, una violación de responsabilidad única a nivel de arquitectura frontend.
