# Estado de cliente frente a estado de servidor

## En una frase

El estado de cliente (un modal abierto, el tema oscuro) **lo decides tú en el navegador**; el estado de servidor (la lista de pedidos) **es una copia de algo que vive en el backend** y puede quedar desactualizada, así que necesita caché, revalidación, reintentos e invalidación, que es lo que resuelven TanStack Query y SWR.

-----

## Antes de empezar

Conviene que ya sepas:

* React con `useState`, `useEffect` y props.
* Cómo responde una API REST y qué son los códigos de estado: [Estándares RESTful](../03-diseno-de-apis-y-contratos/01-estandares-restful.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Estado de cliente:** datos cuyo dueño es el navegador, como un formulario sin enviar o un panel abierto.
* **Estado de servidor:** datos cuyo dueño es el backend; el navegador solo tiene una copia que puede quedar vieja.
* **Caché:** copia guardada de una respuesta para no volver a pedirla.
* **Stale (obsoleto):** un dato en caché que se considera "viejo" y conviene revalidar.
* **Stale-while-revalidate:** mostrar el dato en caché de inmediato mientras se pide uno fresco en segundo plano.
* **Query key:** el identificador de una consulta en la caché, como `["pedidos", userId]`.
* **Invalidación:** marcar una entrada de la caché como obsoleta para que se vuelva a pedir.
* **Backoff exponencial:** esperar cada vez más tiempo entre reintentos: 1 s, 2 s, 4 s, 8 s.
* **SSR (*Server-Side Rendering*):** generar el HTML en el servidor, con los datos ya incluidos, en el momento de la petición.
* **CSR (*Client-Side Rendering*):** el servidor envía un HTML casi vacío y JavaScript arma la página en el navegador.
* **Hidratación:** el proceso por el que React conecta el JavaScript al HTML que llegó del servidor.

-----

## El problema

Un dashboard de pedidos guarda todo en un único store de Redux:

```tsx
// store global con todo mezclado
const initialState = {
  tema: "oscuro",
  modalNuevoPedidoAbierto: false,
  usuario: null,
  pedidos: [],
  pedidosCargando: false,
  pedidosError: null,
};

function ListaPedidos() {
  const dispatch = useDispatch();
  const { pedidos, pedidosCargando } = useSelector((s) => s);

  useEffect(() => {
    dispatch({ type: "pedidos/cargando" });
    fetch("/api/v1/orders")
      .then((r) => r.json())
      .then((data) => dispatch({ type: "pedidos/ok", payload: data }))
      .catch((e) => dispatch({ type: "pedidos/error", payload: e }));
  }, [dispatch]);
  // ...
}
```

Los síntomas en producción:

* el encabezado y la lista piden `/api/v1/orders` al mismo tiempo: dos peticiones idénticas;
* después de crear un pedido, la lista no lo muestra hasta recargar la página;
* si otro usuario cancela un pedido, la pantalla sigue mostrándolo como activo durante horas;
* en el metro, con la conexión intermitente, cualquier error deja la pantalla rota sin reintentar;
* cada pantalla nueva repite el mismo trío de acciones `cargando` / `ok` / `error`.

Las consecuencias: se mezclaron dos tipos de estado con necesidades opuestas, y se reimplementó a mano (y mal) caché, deduplicación, invalidación y reintentos.

-----

## Cómo funciona

### 1. Dos tipos de estado con necesidades distintas

| | Estado de cliente | Estado de servidor |
| --- | --- | --- |
| **Dueño de la verdad** | El navegador: tú decides el valor | El backend: el navegador solo tiene una copia |
| **Ejemplos** | Tema oscuro o claro, un modal abierto, un formulario sin enviar | Lista de productos, perfil del usuario, resultado de una búsqueda |
| **¿Puede quedar desactualizado?** | No: tú lo controlas | Sí: alguien más puede cambiarlo en el servidor |
| **Es asíncrono** | No | Sí: cargando, éxito, error |
| **Herramienta típica** | `useState`, Context, Zustand, Redux | TanStack Query, SWR |

El estado de servidor necesita cosas que un store normal no resuelve:

* guardar la respuesta en caché;
* saber cuándo está obsoleta;
* reintentar si la petición falla;
* no pedir lo mismo dos veces si dos componentes lo necesitan a la vez;
* sincronizarse en segundo plano.

Para eso existen **TanStack Query** (antes React Query) y **SWR**.

> **En una frase:** si para conseguir el dato hay que hacer una petición, es estado de servidor y no va en tu store global.

### 2. Stale-while-revalidate

Las dos librerías se basan en la misma idea del caché HTTP: **stale-while-revalidate**. Muestran el dato en caché de inmediato (aunque esté obsoleto) mientras lo revalidan en segundo plano, en lugar de mostrar una pantalla de carga cada vez.

```text
1ª visita     →  cargando...  →  datos (se guardan en caché)
2ª visita     →  datos de caché al instante  →  (revalida en segundo plano)  →  datos frescos si cambiaron
```

### 3. Caché por clave y deduplicación

Cada consulta se identifica con una **query key**. Si dos componentes piden la misma key, la librería **deduplica**: hace una sola petición y entrega el resultado a los dos.

```tsx
import { useQuery } from "@tanstack/react-query";

function usePedidos(estado: string) {
  return useQuery({
    queryKey: ["pedidos", { estado }],
    queryFn: () => fetch(`/api/v1/orders?status=${estado}`).then((r) => r.json()),
  });
}

function ListaPedidos() {
  const { data, isPending, isError } = usePedidos("activo");

  if (isPending) return <Skeleton />;
  if (isError) return <MensajeError />;
  return <Tabla pedidos={data} />;
}
```

La key es un array porque las consultas dependen de parámetros: `["pedidos", { estado: "activo" }]` y `["pedidos", { estado: "cancelado" }]` son entradas distintas de la caché.

Si lo usas mal:

```tsx
useQuery({
  queryKey: ["pedidos"],
  queryFn: () => fetch(`/api/v1/orders?status=${estado}`).then((r) => r.json()),
});
```

Si un parámetro que usa la `queryFn` no está en la key, cambiar `estado` **no** dispara una nueva consulta: la caché devuelve la lista del estado anterior. Regla: todo lo que usa la `queryFn` va en la `queryKey`. El plugin de ESLint de TanStack Query (`@tanstack/eslint-plugin-query`) lo detecta.

### 4. Invalidación después de una mutación

Después de crear, actualizar o borrar algo, le dices a la librería "esta key ya no es válida", y ella vuelve a pedir el dato fresco. No actualizas el estado local a mano.

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";

function useCrearPedido() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (nuevo: NuevoPedido) =>
      fetch("/api/v1/orders", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(nuevo),
      }).then((r) => r.json()),
    onSuccess: () => {
      // invalida todas las keys que empiezan con "pedidos"
      queryClient.invalidateQueries({ queryKey: ["pedidos"] });
    },
  });
}
```

`invalidateQueries({ queryKey: ["pedidos"] })` invalida por **prefijo**: marca como obsoletas `["pedidos", { estado: "activo" }]`, `["pedidos", { estado: "cancelado" }]` y cualquier otra que empiece con `"pedidos"`.

### 5. Reintentos con backoff exponencial

Si una petición falla (por ejemplo, por un timeout de red), la librería reintenta antes de mostrar un error: TanStack Query hace **3 reintentos** por defecto en el navegador.

**¿Qué es el backoff?** Esperar cada vez más tiempo entre un reintento y el siguiente, en lugar de reintentar de inmediato una y otra vez. "Exponencial" significa que la espera crece en cada intento fallido: 1 s → 2 s → 4 s → 8 s. Sin esto, un cliente que reintenta sin pausa contra un servidor caído o sobrecargado lo empeora (lo satura más), y en móviles agota batería y datos sin necesidad.

> **Ojo:** reintentar tiene sentido para errores **transitorios** (red, `503`), no para un `404` o un `400`, que van a fallar igual en el quinto intento. Se configura con una función, por ejemplo `retry: (count, error) => error instanceof HttpError && error.status >= 500 && count < 3`, donde `HttpError` es una clase de error propia que guarda el código de estado (un `Error` común no lo tiene).

### 6. Revalidación en eventos

Por defecto, las dos librerías revalidan cuando:

* la ventana recupera el foco (`refetchOnWindowFocus`);
* se recupera la conexión a internet (`refetchOnReconnect`);
* se monta un componente que usa una key obsoleta.

Así el usuario no ve datos viejos por mucho tiempo sin darse cuenta.

### 7. `staleTime` y `gcTime`: los dos tiempos que más se confunden

| Opción | Qué controla | Valor por defecto |
| --- | --- | --- |
| `staleTime` | Cuánto tiempo un dato se considera **fresco** (no se revalida) | `0`: obsoleto apenas llega |
| `gcTime` | Cuánto tiempo se guarda en caché un dato que **nadie está usando** | 5 minutos |

Con `staleTime: 0`, cada montaje o cada vez que la ventana recupera el foco dispara una revalidación. Para datos que cambian poco (un catálogo, la configuración), un `staleTime` de varios minutos ahorra muchas peticiones.

### 8. Hidratación limpia con SSR

**¿Qué es SSR?** Server-Side Rendering: el HTML se genera en el servidor cuando el usuario pide la página, con los datos ya incluidos. En CSR, el servidor envía un HTML casi vacío y JavaScript arma todo en el navegador.

*Para no confundir siglas en este roadmap:* el "SSR" del título del repositorio (Semi-Senior) es una sigla distinta de la de Server-Side Rendering. Coinciden en las letras, no en el significado.

En una aplicación con SSR (Next.js), el servidor ya pidió los datos para generar el HTML. Si el cliente vuelve a pedirlos apenas carga la página, aparece un parpadeo (datos → cargando → datos) que no debería existir.

La solución es que el cliente **herede la caché del servidor**:

* `initialData`: le pasas el dato a `useQuery`. Simple, para un caso puntual.
* `dehydrate` + `HydrationBoundary` (TanStack Query v5): el servidor llena un `QueryClient`, lo serializa, y el cliente lo restaura. Escala a varias consultas.

Esto es **hidratación limpia**: el cliente retoma exactamente donde el servidor dejó el estado, sin refetch innecesario ni parpadeo.

> **Ojo con el `staleTime`:** si heredas la caché del servidor pero el `staleTime` es `0`, el dato llega "ya obsoleto" y el cliente lo revalida apenas se monta: se hace la petición que querías evitar. Con SSR, usa un `staleTime` mayor que cero (por ejemplo, 60 segundos).

### 9. ¿Siempre hace falta TanStack Query?

No. Con Server Components de Next.js (ver la [siguiente lección](02-server-client-components.md)), muchas pantallas que solo **muestran** datos los pueden obtener directamente en el servidor, sin ninguna librería en el cliente. TanStack Query aporta más cuando hay **interactividad sobre datos del servidor**: filtros que cambian en vivo, paginación, mutaciones con invalidación, actualizaciones optimistas o datos que tienen que mantenerse frescos mientras el usuario está en la pantalla.

### 10. Conexión con el resto del roadmap

Separar estado de cliente y de servidor es el mismo principio de separar responsabilidades de [SOLID](../02-calidad-de-codigo-y-arquitectura/01-solid-y-patrones.md), aplicado a datos en lugar de a clases: mezclar los dos tipos de estado en un solo store es, en la práctica, una violación de responsabilidad única a nivel de arquitectura frontend.

-----

## Ejemplo completo

El dashboard del problema, en Next.js (App Router) con TanStack Query v5.

`app/providers.tsx` (un `QueryClient` por usuario en el navegador):

```tsx
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState } from "react";

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () => new QueryClient({ defaultOptions: { queries: { staleTime: 60_000 } } })
  );
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
}
```

`lib/pedidos.ts` (las consultas en un solo lugar):

```ts
export const pedidosKeys = {
  todos: ["pedidos"] as const,
  lista: (estado: string) => ["pedidos", { estado }] as const,
};

export async function obtenerPedidos(estado: string): Promise<Pedido[]> {
  const r = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/orders?status=${estado}`);
  if (!r.ok) throw new Error(`Error ${r.status}`);
  return r.json();
}
```

`app/pedidos/page.tsx` (Server Component: precarga en el servidor):

```tsx
import { dehydrate, HydrationBoundary, QueryClient } from "@tanstack/react-query";
import { obtenerPedidos, pedidosKeys } from "@/lib/pedidos";
import { ListaPedidos } from "./lista-pedidos";

export default async function PaginaPedidos() {
  const queryClient = new QueryClient();
  await queryClient.prefetchQuery({
    queryKey: pedidosKeys.lista("activo"),
    queryFn: () => obtenerPedidos("activo"),
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ListaPedidos />
    </HydrationBoundary>
  );
}
```

`app/pedidos/lista-pedidos.tsx` (Client Component: interactividad):

```tsx
"use client";

import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { useState } from "react";
import { obtenerPedidos, pedidosKeys } from "@/lib/pedidos";

export function ListaPedidos() {
  const [estado, setEstado] = useState("activo");                 // estado de CLIENTE
  const [modalAbierto, setModalAbierto] = useState(false);       // estado de CLIENTE
  const queryClient = useQueryClient();

  const pedidos = useQuery({                                     // estado de SERVIDOR
    queryKey: pedidosKeys.lista(estado),
    queryFn: () => obtenerPedidos(estado),
  });

  const cancelar = useMutation({
    mutationFn: (id: number) => fetch(`/api/v1/orders/${id}/cancel`, { method: "POST" }),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: pedidosKeys.todos }),
  });

  if (pedidos.isPending) return <Skeleton />;
  if (pedidos.isError) return <MensajeError onReintentar={() => pedidos.refetch()} />;

  return (
    <>
      <FiltroEstado valor={estado} onCambio={setEstado} />
      <Tabla pedidos={pedidos.data} onCancelar={(id) => cancelar.mutate(id)} />
      <button onClick={() => setModalAbierto(true)}>Nuevo pedido</button>
      {modalAbierto && <ModalNuevoPedido onCerrar={() => setModalAbierto(false)} />}
    </>
  );
}
```

Qué observar:

* El store de Redux desapareció: el tema y el modal son `useState` o Context; los pedidos son de TanStack Query.
* El HTML llega con los pedidos activos; el cliente hereda la caché y, con `staleTime: 60_000`, no vuelve a pedirlos al hidratar.
* Si el encabezado también usa `pedidosKeys.lista("activo")`, se hace una sola petición.
* Cancelar un pedido invalida todas las listas de pedidos, de cualquier estado.
* Al volver a la pestaña o recuperar la conexión, la lista se revalida sola.
* `pedidosKeys` centraliza las keys: un error de tipeo en una key es un bug silencioso de caché.

-----

## Errores comunes

**1. Guardar estado de servidor en un store global.**
Qué pasa: peticiones duplicadas, datos viejos y la misma lógica de cargando/error repetida en cada pantalla.
Por qué: se trata la respuesta de la API como si fuera estado de cliente.
Arreglo: TanStack Query o SWR para todo lo que viene del backend; el store, solo para estado de cliente.

**2. Parámetros que no están en la `queryKey`.**
Qué pasa: cambias un filtro y la lista sigue mostrando lo anterior.
Por qué: la caché identifica la consulta por la key, y la key no cambió.
Arreglo: todo lo que usa la `queryFn` va en la `queryKey`; activa el plugin de ESLint.

**3. Actualizar la caché a mano después de cada mutación.**
Qué pasa: la lista local se desincroniza del servidor (falta un campo calculado, el orden es distinto).
Por qué: se copia en el cliente la lógica que ya hace el backend.
Arreglo: `invalidateQueries` después de la mutación; actualiza a mano solo cuando busques una actualización optimista a propósito.

**4. Un `QueryClient` compartido en el servidor.**
Qué pasa: un usuario ve datos de otro.
Por qué: un `QueryClient` creado a nivel de módulo en el servidor se comparte entre todas las peticiones.
Arreglo: en el servidor, un `QueryClient` nuevo por petición; en el navegador, uno por aplicación (con `useState`, como en el ejemplo).

**5. SSR con `staleTime: 0`.**
Qué pasa: los datos se piden dos veces: en el servidor y apenas hidrata el cliente.
Por qué: el dato heredado llega ya obsoleto.
Arreglo: `staleTime` mayor que cero cuando hay SSR.

**6. Usar `fetch` sin revisar `response.ok`.**
Qué pasa: un `500` llega como "datos" y la pantalla falla al renderizar.
Por qué: `fetch` solo rechaza la promesa por errores de red, no por códigos HTTP de error.
Arreglo: `if (!r.ok) throw new Error(...)` en la `queryFn`, para que la librería lo trate como error y reintente.

-----

## Evolución y versiones

* **2019:** Tanner Linsley publica React Query; Vercel publica SWR.
* **2022:** React Query v4 pasa a llamarse **TanStack Query**, con soporte para Vue, Svelte y Solid.
* **2023:** TanStack Query v5: `cacheTime` pasa a llamarse `gcTime`, `isLoading` pasa a `isPending`, una sola firma de objeto para `useQuery`, y `Hydrate` pasa a `HydrationBoundary`.

Si sigues un tutorial anterior a 2023, vas a ver `useQuery(["pedidos"], fn)` y `cacheTime`: es la API de la v4.

-----

## Cuándo sí y cuándo no

**Usa TanStack Query o SWR cuando:** la pantalla consume datos de una API y hay interactividad sobre ellos: filtros, paginación, mutaciones, datos que tienen que mantenerse frescos.

**Usa un store de cliente (Zustand, Redux, Context) cuando:** el estado es de la interfaz y vive solo en el navegador: tema, panel abierto, pasos de un asistente, un carrito antes de enviarlo.

**No uses TanStack Query cuando:** la pantalla solo muestra datos que se pueden obtener en un Server Component, sin interactividad sobre ellos.

-----

## Resumen en 5 líneas

1. El estado de cliente lo decide el navegador; el de servidor es una copia que puede quedar vieja.
2. El estado de servidor necesita caché, deduplicación, revalidación, reintentos e invalidación: TanStack Query o SWR.
3. Todo lo que usa la `queryFn` va en la `queryKey`; después de una mutación, invalida.
4. `staleTime` dice cuándo un dato deja de ser fresco; `gcTime`, cuánto se guarda sin uso.
5. Con SSR, el cliente hereda la caché del servidor (`HydrationBoundary`) y usa un `staleTime` mayor que cero.

-----

## Para profundizar

<details>
<summary>Actualizaciones optimistas</summary>

En lugar de esperar la respuesta del servidor, actualizas la caché al instante, como si la operación ya hubiera funcionado, y la reviertes si falla:

```tsx
useMutation({
  mutationFn: cancelarPedido,
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: pedidosKeys.todos });
    const anterior = queryClient.getQueryData<Pedido[]>(pedidosKeys.lista("activo"));
    queryClient.setQueryData<Pedido[]>(pedidosKeys.lista("activo"), (p) => p?.filter((x) => x.id !== id));
    return { anterior };
  },
  onError: (_err, _id, contexto) =>
    queryClient.setQueryData(pedidosKeys.lista("activo"), contexto?.anterior),
  onSettled: () => queryClient.invalidateQueries({ queryKey: pedidosKeys.todos }),
});
```

La interfaz se siente instantánea. Úsalo para acciones que casi nunca fallan (marcar como favorito, cancelar); para un pago, espera la confirmación.

</details>

<details>
<summary>TanStack Query frente a SWR</summary>

Las dos implementan stale-while-revalidate. **SWR** es más chica y simple, ideal si solo necesitas leer y revalidar. **TanStack Query** trae más herramientas integradas: mutaciones con invalidación, actualizaciones optimistas, consultas paginadas e infinitas, DevTools y control fino de reintentos y tiempos. Para una aplicación con muchas mutaciones, TanStack Query suele ahorrar más código.

</details>

-----

## En entrevista

### Respuesta corta (junior)

El estado de cliente es lo que controla la interfaz, como un modal abierto, y se maneja con `useState` o un store. El estado de servidor son los datos que vienen de la API, y conviene manejarlos con TanStack Query o SWR, que se ocupan de la caché, la carga y los errores.

### Respuesta ampliada (semi-senior)

Los separo porque tienen necesidades opuestas: el estado de servidor es una copia que puede quedar vieja, así que necesita caché, deduplicación, revalidación, reintentos con backoff e invalidación después de las mutaciones. Para eso uso TanStack Query, con todo lo que usa la `queryFn` dentro de la `queryKey` e invalidación por prefijo. El store global lo dejo solo para estado de interfaz. Con Next.js y SSR, precargo en el servidor y el cliente hereda la caché con `HydrationBoundary`, con un `staleTime` mayor que cero para no volver a pedir al hidratar, y un `QueryClient` por petición en el servidor para no mezclar datos de usuarios. Y si una pantalla solo muestra datos, a veces alcanza con un Server Component.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre `staleTime` y `gcTime`?**
`staleTime` es cuánto tiempo un dato se considera fresco y no se revalida. `gcTime` es cuánto tiempo se guarda en caché un dato que ningún componente está usando antes de borrarlo.

**2. ¿Por qué no guardar la respuesta de la API en Redux?**
Porque hay que reimplementar a mano la caché, la deduplicación, la revalidación, los reintentos y la invalidación, y el dato puede quedar viejo sin que nadie lo note.

**3. ¿Qué es stale-while-revalidate?**
Mostrar el dato en caché de inmediato, aunque esté obsoleto, mientras se pide uno fresco en segundo plano.

-----

## Práctica

**Ejercicio 1.** Clasifica cada dato como estado de cliente o de servidor, y di con qué lo manejarías.

1. Si el menú lateral está colapsado.
2. El catálogo de productos.
3. El texto que el usuario está escribiendo en un buscador.
4. Los resultados de esa búsqueda.
5. El perfil del usuario que inició sesión.
6. El paso actual de un asistente de registro de 4 pasos.

<details>
<summary>Solución</summary>

1. **Cliente:** `useState` o un store (si varias pantallas lo necesitan).
2. **Servidor:** TanStack Query, con un `staleTime` alto porque cambia poco.
3. **Cliente:** `useState`, idealmente con *debounce* antes de buscar.
4. **Servidor:** TanStack Query con el texto en la key: `["busqueda", texto]`.
5. **Servidor:** TanStack Query. Que "se use en toda la app" no lo convierte en estado de cliente.
6. **Cliente:** `useState` o un store del asistente.

</details>

**Ejercicio 2.** ¿Qué tiene de malo este código? Corrígelo.

```tsx
function Productos({ categoria }: { categoria: string }) {
  const { data } = useQuery({
    queryKey: ["productos"],
    queryFn: () => fetch(`/api/v1/products?category=${categoria}`).then((r) => r.json()),
  });

  const crear = useMutation({
    mutationFn: crearProducto,
    onSuccess: (nuevo) => data.push(nuevo),
  });
  // ...
}
```

<details>
<summary>Solución</summary>

* `categoria` no está en la `queryKey`: al cambiar de categoría se sigue mostrando la anterior.
* `fetch` no revisa `r.ok`: un `500` se trata como datos.
* `data.push(nuevo)` muta el objeto de la caché directamente: React no se entera y el componente no se vuelve a renderizar, y además el nuevo producto puede no pertenecer a esa categoría.

```tsx
function Productos({ categoria }: { categoria: string }) {
  const queryClient = useQueryClient();

  const { data } = useQuery({
    queryKey: ["productos", { categoria }],
    queryFn: async () => {
      const r = await fetch(`/api/v1/products?category=${categoria}`);
      if (!r.ok) throw new Error(`Error ${r.status}`);
      return r.json();
    },
  });

  const crear = useMutation({
    mutationFn: crearProducto,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ["productos"] }),
  });
  // ...
}
```

</details>

-----

## Siguiente lección

[Server y Client Components](02-server-client-components.md)
