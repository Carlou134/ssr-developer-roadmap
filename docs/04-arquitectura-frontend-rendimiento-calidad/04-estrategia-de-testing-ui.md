# Estrategia de testing en UI

## En una frase

Una buena prueba de UI **usa el componente como lo usaría una persona**: lo busca por su rol y su texto, interactúa con eventos reales, recibe datos de una red simulada con MSW y verifica los cuatro estados (cargando, éxito, vacío y error), no solo el camino feliz.

-----

## Antes de empezar

Conviene que ya sepas:

* La pirámide de testing y la cobertura útil: [Estrategia de testing](../02-calidad-de-codigo-y-arquitectura/03-estrategia-de-testing.md).
* Estado de servidor y TanStack Query: [Estado de cliente frente a estado de servidor](01-estado-cliente-vs-servidor.md).
* Componentes de React con `useState` y eventos.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Test runner:** herramienta que encuentra, ejecuta y reporta las pruebas, como Jest o Vitest.
* **React Testing Library (RTL):** librería para probar componentes de React desde el punto de vista del usuario.
* **Query (de RTL):** función para buscar elementos en la pantalla, como `getByRole`.
* **Rol accesible (ARIA role):** lo que un elemento **es** para las tecnologías de asistencia: botón, campo de texto, encabezado.
* **`user-event`:** librería que simula interacciones reales del usuario (escribir, hacer clic, tabular).
* **MSW (*Mock Service Worker*):** librería que intercepta peticiones HTTP a nivel de red y responde con datos simulados.
* **Handler (de MSW):** la definición de qué responder para una ruta concreta.
* **jsdom:** implementación del DOM en Node, para ejecutar pruebas de UI sin un navegador.
* **Prueba frágil:** prueba que se rompe con un refactor aunque el comportamiento no haya cambiado.

-----

## El problema

El componente `ListaPedidos` "tiene 100 % de cobertura":

```tsx
it("renderiza pedidos", async () => {
  jest.spyOn(axios, "get").mockResolvedValue({ data: [{ id: 1, total: 100 }] });
  const { container } = render(<ListaPedidos />);

  await waitFor(() => {
    expect(container.querySelector(".pedido-item")).toBeTruthy();
  });
  expect(wrapper.state("pedidos")).toHaveLength(1); // detalle interno
});
```

Lo que pasa después:

* el equipo cambia `axios` por `fetch` y las pruebas fallan, aunque la pantalla funciona igual;
* alguien renombra la clase CSS `.pedido-item` a `.fila-pedido` y otra prueba falla;
* en producción, cuando la API tarda 3 segundos, la pantalla queda en blanco: nunca se probó el estado de carga;
* cuando el usuario no tiene pedidos, ve una tabla vacía sin explicación;
* cuando la API devuelve `500`, la aplicación se rompe con `Cannot read properties of undefined`.

Las consecuencias: las pruebas fallan cuando nada está roto y pasan cuando todo lo está.

-----

## Cómo funciona

### 1. Jest y Vitest

Los dos son **test runners** con una API casi idéntica (`describe`, `it`, `expect`, `vi.fn()` / `jest.fn()`), así que lo que aprendes en uno se transfiere al otro.

| | Jest | Vitest |
| --- | --- | --- |
| Origen | El histórico (Meta, 2014) | Construido sobre Vite (2021) |
| Módulos | CommonJS por defecto; ESM con configuración | ESM nativo |
| Velocidad | Buena | Notablemente más rápido en proyectos con Vite, porque reutiliza su configuración y su transformación |
| Configuración | Propia (Babel o `ts-jest` para TypeScript) | Reutiliza `vite.config` |

Para un proyecto nuevo con Vite, Vitest es hoy la opción por defecto. Next.js soporta los dos. En un proyecto que ya usa Jest y funciona bien, migrar no es obligatorio: la ganancia de velocidad no siempre justifica el costo.

> **Contexto:** muchos tutoriales usan Jest porque venía integrado en **Create React App**, que el equipo de React dejó de recomendar y marcó como deprecado en febrero de 2025. Para proyectos nuevos se usa Vite o un framework como Next.js.

### 2. La filosofía de React Testing Library

El lema de RTL resume la idea: **"cuanto más se parezcan tus pruebas a la forma en que se usa el software, más confianza te dan"**.

La consecuencia práctica: RTL evita a propósito que accedas a detalles internos (el estado de un componente, sus métodos, sus clases CSS). En su lugar, buscas los elementos como lo haría un usuario.

Si una prueba depende de un detalle interno, refactorizar el componente sin cambiar su comportamiento la rompe aunque nada esté roto. Eso es una **prueba frágil**, no una prueba útil.

### 3. Cómo buscar elementos: prioridad de las queries

| Prioridad | Query | Cuándo |
| --- | --- | --- |
| 1 | `getByRole("button", { name: "Guardar" })` | La forma preferida para casi todo |
| 2 | `getByLabelText("Email")` | Campos de formulario, por su label |
| 3 | `getByPlaceholderText`, `getByText`, `getByDisplayValue` | Contenido visible |
| 4 | `getByAltText`, `getByTitle` | Imágenes y elementos con título |
| 5 | `getByTestId("grafico")` | **Último recurso**, cuando no hay forma semántica de identificarlo |

`getByRole` es la preferida porque verifica accesibilidad de forma indirecta: si un botón no tiene un rol accesible ni un nombre, la prueba no lo encuentra, y un lector de pantalla tampoco. Abusar de `data-testid` es una señal de que el HTML no es lo bastante semántico (ver [Accesibilidad](../09-fundamentos-diseno-ui-ux/10-accesibilidad.md)).

Además de la prioridad, cada query tiene tres variantes según **qué** esperas:

| Variante | Si no encuentra el elemento | Úsala para |
| --- | --- | --- |
| `getBy...` | Lanza un error | Elementos que **deben** estar ya |
| `queryBy...` | Devuelve `null` | Verificar que algo **no** está: `expect(screen.queryByText("Error")).not.toBeInTheDocument()` |
| `findBy...` | Espera (hasta 1 s por defecto) y devuelve una promesa | Elementos que aparecen **después**: tras una petición o una animación |

Si lo usas mal:

```tsx
render(<ListaPedidos />);
expect(screen.getByText("Pedido #1")).toBeInTheDocument();
```

```text
TestingLibraryElementError: Unable to find an element with the text: Pedido #1
```

El pedido llega después de una petición asíncrona: en el momento del `getBy`, todavía se muestra el estado de carga. Para lo que aparece después, usa `await screen.findByText(...)`.

### 4. Interactuar como el usuario: `user-event`

```tsx
import userEvent from "@testing-library/user-event";

const user = userEvent.setup();
await user.type(screen.getByLabelText("Buscar cliente"), "Ana");
await user.click(screen.getByRole("button", { name: "Filtrar" }));
```

| | `fireEvent` | `user-event` |
| --- | --- | --- |
| Qué hace | Dispara **un** evento del DOM | Simula la interacción completa: foco, teclas, `input`, `change`, `click` |
| Ejemplo | `fireEvent.change(input, { target: { value: "Ana" } })` | `await user.type(input, "Ana")` |
| Detecta | Poco | Botones deshabilitados, elementos tapados, campos de solo lectura |

Usa `user-event` por defecto: un `fireEvent.click` sobre un botón deshabilitado dispara el evento igual, aunque un usuario real no podría hacerlo.

### 5. Simular la red con MSW

Para probar un componente que pide datos a una API hay dos formas de reemplazar la llamada:

| | Mockear `fetch` o `axios` | MSW |
| --- | --- | --- |
| Cómo | `vi.spyOn(globalThis, "fetch")` o `jest.mock("axios")` | Intercepta la petición a nivel de red |
| Qué acopla la prueba a... | **Cómo** el componente pide (qué librería usa) | **Qué** responde el servidor |
| Si cambias `axios` por `fetch` | La prueba se rompe | La prueba sigue igual |
| Reutilizable en desarrollo | No | Sí: los mismos handlers sirven para trabajar sin backend |

MSW funciona con un Service Worker en el navegador y con interceptores de los módulos `http` y `https` en Node. El componente hace un `fetch` real, MSW lo intercepta antes de que salga y responde con los datos que definiste.

```ts
// mocks/handlers.ts (MSW v2)
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/v1/orders", () =>
    HttpResponse.json([
      { id: 1, cliente: "Ana", total: 120 },
      { id: 2, cliente: "Luis", total: 80 },
    ])
  ),
];
```

```ts
// test/setup.ts
import { setupServer } from "msw/node";
import { handlers } from "../mocks/handlers";

export const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

`onUnhandledRequest: "error"` hace fallar la prueba si el componente pide una ruta que no tiene handler: así no hay peticiones reales "escapadas" a internet.

> **Si sigues un tutorial viejo:** MSW v1 usaba `rest.get(...)` y `res(ctx.json(...))`. La v2 (2023) cambió a `http.get(...)` y `HttpResponse.json(...)`, alineada con la API estándar de `fetch`.

### 6. Probar los cuatro estados, no solo el camino feliz

Un componente que consume una API tiene, como mínimo, cuatro estados, y cada uno merece su prueba:

| # | Estado | Qué verificar |
| --- | --- | --- |
| 1 | **Cargando** | Se muestra un spinner o skeleton mientras espera |
| 2 | **Éxito con datos** | Los datos se renderizan correctamente |
| 3 | **Éxito sin datos** | Con una lista vacía, un mensaje claro ("No tienes pedidos"), no una pantalla en blanco |
| 4 | **Error** | Si la API falla, un mensaje claro y, si corresponde, una forma de reintentar |

Probar solo el estado 2 es la razón más común por la que un componente "pasa todas las pruebas" en CI pero falla en producción la primera vez que la API tarda, devuelve vacío o se cae. Es el mismo argumento de la [cobertura útil](../02-calidad-de-codigo-y-arquitectura/03-estrategia-de-testing.md) de la fase 2: cubrir líneas no es cubrir comportamientos.

Con MSW, cada estado es un handler distinto que se reemplaza en esa prueba con `server.use(...)`.

### 7. Qué probar en cada nivel

| Nivel | Herramienta | Qué prueba |
| --- | --- | --- |
| Unitarias | Vitest o Jest | Funciones puras y custom hooks con lógica (formateo, validaciones, cálculos) |
| Componentes | RTL + MSW | Un componente o una pantalla con sus estados e interacciones |
| E2E | Playwright o Cypress | Flujos críticos en un navegador real, de punta a punta |

> **Ojo con los Server Components asíncronos:** Vitest y Jest **no** pueden renderizar Server Components `async` de Next.js. La recomendación oficial es probar la lógica de esos componentes por separado y cubrir la página completa con pruebas E2E (Playwright).

-----

## Ejemplo completo

`ListaPedidos` con los cuatro estados probados.

`lista-pedidos.tsx`:

```tsx
"use client";

import { useQuery } from "@tanstack/react-query";

async function obtenerPedidos(): Promise<Pedido[]> {
  const r = await fetch("/api/v1/orders");
  if (!r.ok) throw new Error(`Error ${r.status}`);
  return r.json();
}

export function ListaPedidos() {
  const { data, isPending, isError, refetch } = useQuery({
    queryKey: ["pedidos"],
    queryFn: obtenerPedidos,
  });

  if (isPending) return <p role="status">Cargando pedidos...</p>;

  if (isError)
    return (
      <div role="alert">
        <p>No pudimos cargar tus pedidos.</p>
        <button onClick={() => refetch()}>Reintentar</button>
      </div>
    );

  if (data.length === 0) return <p>Todavía no tienes pedidos.</p>;

  return (
    <ul aria-label="Pedidos">
      {data.map((p) => (
        <li key={p.id}>
          {p.cliente}: ${p.total}
        </li>
      ))}
    </ul>
  );
}
```

`test/render-con-query.tsx` (cada prueba con su propio `QueryClient`, sin reintentos):

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { render } from "@testing-library/react";

export function renderConQuery(ui: React.ReactElement) {
  const queryClient = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>);
}
```

`lista-pedidos.test.tsx`:

```tsx
import { screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { http, HttpResponse, delay } from "msw";
import { server } from "../test/setup";
import { renderConQuery } from "../test/render-con-query";
import { ListaPedidos } from "./lista-pedidos";

describe("ListaPedidos", () => {
  it("muestra un estado de carga mientras espera", async () => {
    server.use(http.get("/api/v1/orders", async () => {
      await delay("infinite");
      return HttpResponse.json([]);
    }));

    renderConQuery(<ListaPedidos />);

    expect(screen.getByRole("status")).toHaveTextContent("Cargando pedidos");
  });

  it("muestra los pedidos cuando la API responde", async () => {
    renderConQuery(<ListaPedidos />); // usa el handler por defecto: Ana y Luis

    const lista = await screen.findByRole("list", { name: "Pedidos" });
    expect(within(lista).getAllByRole("listitem")).toHaveLength(2);
    expect(within(lista).getByText(/Ana/)).toBeInTheDocument();
  });

  it("muestra un mensaje cuando no hay pedidos", async () => {
    server.use(http.get("/api/v1/orders", () => HttpResponse.json([])));

    renderConQuery(<ListaPedidos />);

    expect(await screen.findByText("Todavía no tienes pedidos.")).toBeInTheDocument();
  });

  it("muestra un error y permite reintentar", async () => {
    server.use(http.get("/api/v1/orders", () => new HttpResponse(null, { status: 500 }), { once: true }));
    const user = userEvent.setup();

    renderConQuery(<ListaPedidos />);

    const alerta = await screen.findByRole("alert");
    expect(alerta).toHaveTextContent("No pudimos cargar tus pedidos");

    await user.click(within(alerta).getByRole("button", { name: "Reintentar" }));

    expect(await screen.findByRole("list", { name: "Pedidos" })).toBeInTheDocument();
  });
});
```

Ejecución:

```text
npx vitest run
```

Qué observar:

* Ninguna prueba sabe si el componente usa `fetch`, `axios` o TanStack Query: solo ven la pantalla y la red.
* Todas las búsquedas son por rol o por texto; no hay clases CSS ni `data-testid`.
* `role="status"` y `role="alert"` hacen que el componente sea más accesible **y** más fácil de probar.
* Cada prueba crea su `QueryClient`: no comparten caché entre ellas.
* `retry: false` evita que la prueba de error espere los tres reintentos con backoff.
* `{ once: true }` hace que el `500` ocurra solo la primera vez, así el reintento recibe la respuesta por defecto.

-----

## Errores comunes

**1. Probar detalles de implementación.**
Qué pasa: las pruebas fallan con cada refactor aunque la pantalla funcione igual.
Por qué: se verifica el estado interno, las clases CSS o qué librería HTTP se usa.
Arreglo: busca por rol y texto, interactúa con `user-event` y verifica lo que ve el usuario.

**2. Usar `getBy` para algo que aparece después.**
Qué pasa: `Unable to find an element` en un componente que funciona.
Por qué: el elemento llega tras una petición asíncrona.
Arreglo: `await screen.findBy...`.

**3. Probar solo el camino feliz.**
Qué pasa: pantallas en blanco o rotas cuando la API tarda, devuelve vacío o falla.
Por qué: los estados de carga, vacío y error nunca se ejercitan.
Arreglo: una prueba por cada uno de los cuatro estados, con un handler de MSW distinto.

**4. Compartir el `QueryClient` entre pruebas.**
Qué pasa: una prueba pasa sola y falla con las demás (o al revés).
Por qué: la caché de una prueba contamina la siguiente.
Arreglo: un `QueryClient` nuevo por prueba, con `retry: false`.

**5. Abusar de `data-testid`.**
Qué pasa: pruebas que pasan aunque el botón no sea accesible para un lector de pantalla.
Por qué: el `data-testid` encuentra el elemento sin importar su semántica.
Arreglo: usa `getByRole`; si no puedes, probablemente el HTML necesita un rol, un label o un nombre.

**6. Peticiones reales que escapan a internet.**
Qué pasa: pruebas lentas e intermitentes que dependen de un servidor externo.
Por qué: falta un handler para alguna ruta y la petición sale de verdad.
Arreglo: `server.listen({ onUnhandledRequest: "error" })`.

-----

## Evolución y versiones

* **2014:** Meta publica Jest.
* **2018:** Kent C. Dodds publica React Testing Library, como alternativa a Enzyme y su enfoque en detalles internos.
* **2021:** aparece Vitest, sobre Vite.
* **2023:** MSW v2 cambia la API a `http` y `HttpResponse`.
* **Febrero de 2025:** Create React App queda deprecado; los proyectos nuevos usan Vite o un framework.

-----

## Cuándo sí y cuándo no

**Usa RTL + MSW cuando:** pruebas componentes o pantallas con datos de una API, formularios o interacciones. Es el nivel que más confianza da por su costo en el frontend.

**Usa pruebas unitarias puras cuando:** la lógica se puede separar del componente (formateo, validaciones, cálculos).

**Usa E2E cuando:** el flujo es crítico (login, pago) o involucra Server Components `async`, navegación entre páginas o el backend real.

**No pruebes:** estilos y detalles visuales con RTL (para eso están las pruebas de regresión visual), ni componentes puramente presentacionales sin lógica.

-----

## Resumen en 5 líneas

1. Prueba como el usuario: busca por rol y texto, interactúa con `user-event`, y deja `data-testid` como último recurso.
2. `getBy` para lo que ya está, `queryBy` para verificar que algo no está, `findBy` para lo que aparece después.
3. MSW simula la red, así las pruebas no dependen de qué librería HTTP usa el componente.
4. Cuatro estados por componente con datos: cargando, éxito, vacío y error.
5. Vitest para proyectos nuevos con Vite; los Server Components `async` se cubren con E2E.

-----

## Para profundizar

<details>
<summary>Probar custom hooks con <code>renderHook</code></summary>

Para un hook con lógica propia, `renderHook` lo ejecuta dentro de un componente de prueba:

```tsx
import { renderHook, act } from "@testing-library/react";

it("useContador incrementa", () => {
  const { result } = renderHook(() => useContador(5));

  act(() => result.current.incrementar());

  expect(result.current.valor).toBe(6);
});
```

Úsalo para hooks reutilizables con lógica no trivial. Si el hook solo envuelve un `useQuery`, es más útil probar el componente que lo usa.

</details>

<details>
<summary>Los mismos handlers de MSW en desarrollo</summary>

Los handlers que usas en las pruebas también sirven en el navegador, con un Service Worker:

```ts
// mocks/browser.ts
import { setupWorker } from "msw/browser";
import { handlers } from "./handlers";

export const worker = setupWorker(...handlers);
```

Arrancado en desarrollo (por ejemplo, cuando una variable de entorno lo indica), permite trabajar en el frontend antes de que el backend exista, o reproducir a mano un caso difícil (un error 500, una respuesta lenta). Encaja con el enfoque *contract-first* de [Documentación técnica](../03-diseno-de-apis-y-contratos/03-documentacion-tecnica.md).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Pruebo componentes de React con React Testing Library y Jest o Vitest. Busco los elementos por su rol o su texto, como lo haría un usuario, y simulo las llamadas a la API con mocks.

### Respuesta ampliada (semi-senior)

Sigo la filosofía de RTL: pruebo comportamiento, no implementación. Busco por rol y label, lo que además me obliga a que el HTML sea accesible, interactúo con `user-event` y uso `findBy` para lo asíncrono. Para la red uso MSW, que intercepta a nivel de red: las pruebas no se rompen si cambio de librería HTTP y los mismos handlers me sirven para desarrollar sin backend. En cada componente con datos pruebo los cuatro estados: cargando, éxito, vacío y error, cambiando el handler en cada prueba, y con un `QueryClient` nuevo y sin reintentos para que no se contaminen. Lo que RTL no puede cubrir, como Server Components asíncronos o flujos críticos entre páginas, lo dejo para E2E con Playwright.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué `getByRole` y no `getByTestId`?**
Porque busca el elemento como lo percibe un usuario o un lector de pantalla. Si no lo encuentra, probablemente el elemento no es accesible. `data-testid` lo encuentra igual, y oculta ese problema.

**2. ¿Por qué MSW y no mockear `fetch`?**
Porque acopla la prueba a lo que responde el servidor y no a cómo el componente hace la petición. Si cambio `axios` por `fetch`, las pruebas siguen funcionando.

**3. ¿Qué diferencia hay entre `getBy`, `queryBy` y `findBy`?**
`getBy` falla si el elemento no está; `queryBy` devuelve `null`, para verificar ausencia; `findBy` espera a que aparezca y devuelve una promesa.

-----

## Práctica

**Ejercicio 1.** Reescribe esta prueba siguiendo la filosofía de RTL.

```tsx
it("envía el formulario", () => {
  const { container } = render(<Login onSubmit={onSubmit} />);
  const inputs = container.querySelectorAll("input");
  fireEvent.change(inputs[0], { target: { value: "ana@mail.com" } });
  fireEvent.change(inputs[1], { target: { value: "secreto" } });
  fireEvent.click(container.querySelector(".btn-primary")!);
  expect(onSubmit).toHaveBeenCalled();
});
```

<details>
<summary>Solución</summary>

```tsx
it("envía el email y la contraseña al iniciar sesión", async () => {
  const onSubmit = vi.fn();
  const user = userEvent.setup();
  render(<Login onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText("Email"), "ana@mail.com");
  await user.type(screen.getByLabelText("Contraseña"), "secreto");
  await user.click(screen.getByRole("button", { name: "Iniciar sesión" }));

  expect(onSubmit).toHaveBeenCalledWith({ email: "ana@mail.com", password: "secreto" });
});
```

* Busca los campos por su label, no por su posición: si se agrega un campo, no se rompe.
* Busca el botón por rol y nombre, no por una clase CSS.
* `user-event` simula la escritura real (y fallaría si el botón estuviera deshabilitado).
* Verifica **con qué datos** se llamó a `onSubmit`, no solo que se llamó.
* Si `getByLabelText` no encuentra los campos, el formulario no tiene labels: es un problema de accesibilidad que la prueba acaba de detectar.

</details>

**Ejercicio 2.** Un componente `PerfilUsuario` pide `/api/v1/me`. Escribe los handlers de MSW y las verificaciones para sus cuatro estados (no hace falta el código completo de cada prueba).

<details>
<summary>Solución</summary>

| Estado | Handler | Verificación |
| --- | --- | --- |
| Cargando | `http.get("/api/v1/me", async () => { await delay("infinite"); return HttpResponse.json({}); })` | `screen.getByRole("status")` |
| Éxito | `http.get("/api/v1/me", () => HttpResponse.json({ nombre: "Ana", email: "ana@mail.com" }))` | `await screen.findByRole("heading", { name: "Ana" })` |
| Vacío | En un perfil no hay "vacío"; el equivalente es `401`: `new HttpResponse(null, { status: 401 })` | `await screen.findByRole("link", { name: "Iniciar sesión" })` |
| Error | `new HttpResponse(null, { status: 500 })` | `await screen.findByRole("alert")` con un botón "Reintentar" |

No todos los componentes tienen un estado "vacío" literal: la pregunta útil es **¿qué respuestas válidas pero no felices puede dar la API?** Para un perfil, "no autenticado" es la más importante.

</details>

-----

## Siguiente lección

Terminaste las lecciones de esta fase. Para pedir ayuda estructurada con lo que aprendiste, usa los [Prompts útiles](05-prompts-utiles.md).
