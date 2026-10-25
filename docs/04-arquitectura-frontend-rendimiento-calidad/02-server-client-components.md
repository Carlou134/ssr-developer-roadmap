# Server y Client Components: de la SPA a la arquitectura híbrida

## En una frase

En Next.js (App Router), cada componente decide dónde se ejecuta: **los Server Components corren solo en el servidor y no envían JavaScript al navegador**, los Client Components (`"use client"`) se hidratan y permiten interactividad; la regla es empezar en el servidor y bajar al cliente solo lo que de verdad necesita hooks o eventos.

-----

## Antes de empezar

Conviene que ya sepas:

* Estado de cliente frente a servidor, SSR e hidratación: [Estado de cliente frente a estado de servidor](01-estado-cliente-vs-servidor.md).
* Componentes, props, `useState` y `useEffect` en React.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **MPA (*Multi-Page Application*):** cada navegación pide al servidor una página HTML completa.
* **SPA (*Single-Page Application*):** el HTML se carga una vez y JavaScript actualiza la pantalla pidiendo solo datos.
* **Server Component:** componente de React que se ejecuta solo en el servidor y no envía su código al navegador.
* **Client Component:** componente que se ejecuta en el navegador (y también se prerenderiza en el servidor); permite hooks y eventos.
* **`"use client"`:** directiva que marca el límite a partir del cual los componentes son de cliente.
* **Hydration mismatch:** el HTML del servidor no coincide con lo que React genera en el cliente al hidratar.
* **Bundle:** el JavaScript que el navegador tiene que descargar para la aplicación.
* **Serializable:** un valor que se puede convertir a texto para enviarlo por la red, como un string, un número o un objeto plano.
* **Custom hook:** función que empieza con `use` y reutiliza lógica con estado entre componentes.
* **Prop drilling:** pasar una prop por varios componentes intermedios que no la usan, solo para que llegue a uno que sí.

-----

## El problema

Un catálogo de productos hecho como SPA pura:

```tsx
"use client"; // en la raíz: toda la app es de cliente

export default function Catalogo() {
  const [productos, setProductos] = useState<Producto[]>([]);
  const [usuario, setUsuario] = useState<Usuario | null>(null);

  useEffect(() => {
    fetch("/api/products").then((r) => r.json()).then(setProductos);
    fetch("/api/me").then((r) => r.json()).then(setUsuario);
  }, []);

  return (
    <Layout usuario={usuario}>
      <Encabezado usuario={usuario}>
        <Menu usuario={usuario}>
          <AvatarUsuario usuario={usuario} />
        </Menu>
      </Encabezado>
      <p>Actualizado: {new Date().toLocaleTimeString()}</p>
      <Grilla productos={productos} />
    </Layout>
  );
}
```

Las consecuencias:

* el navegador descarga 450 KB de JavaScript, incluida la librería de formateo de Markdown que solo usan las descripciones, antes de mostrar nada útil;
* el HTML inicial llega sin productos: el usuario ve una pantalla en blanco y los buscadores indexan una página vacía en la primera pasada;
* `usuario` atraviesa `Layout`, `Encabezado` y `Menu` solo para llegar a `AvatarUsuario`;
* la consola muestra `Text content does not match server-rendered HTML` por la hora.

-----

## Cómo funciona

### 1. De la MPA a la SPA, y sus problemas

Antes de las SPA, cada clic pedía al servidor una página HTML completa (**MPA**). Las **SPA** (React, Vue, Angular) cambiaron eso: el HTML inicial se carga una vez y JavaScript actualiza solo la parte de la pantalla que cambia, pidiendo al servidor solo los datos (JSON).

Mejoró mucho la fluidez, pero trajo dos problemas que arrastra toda SPA pura:

* **SEO y primer pintado:** si el HTML llega casi vacío y el contenido lo pinta JavaScript, el usuario ve una pantalla en blanco unos instantes. Googlebot sí ejecuta JavaScript, pero lo hace en una segunda oleada de rastreo, así que la indexación es más lenta e insegura que con HTML ya armado. Otros rastreadores directamente no lo ejecutan.
* **Carga inicial pesada:** todo el JavaScript de la aplicación se descarga antes de poder interactuar con casi nada.

(El problema del botón "Atrás" que rompían las SPA antiguas ya está resuelto: React Router, Vue Router y los demás manejan el historial con la History API. No es una desventaja vigente).

### 2. Server Components y Client Components

Next.js con App Router no es "volver a las MPA": es un modelo **híbrido**, donde cada componente decide dónde se ejecuta.

| | Server Component (por defecto) | Client Component (`"use client"`) |
| --- | --- | --- |
| Dónde se ejecuta | Solo en el servidor | En el navegador (y se prerenderiza en el servidor para el HTML inicial) |
| JavaScript enviado al navegador | **Ninguno** para ese componente | Su código y el de sus dependencias |
| Puede usar | `async`/`await`, acceso directo a base de datos, secretos, archivos | `useState`, `useEffect`, eventos (`onClick`), APIs del navegador (`window`, `localStorage`) |
| No puede usar | Hooks de estado ni eventos | Acceso directo a la base de datos ni secretos |
| Ideal para | Layouts, listas de datos, contenido, todo lo que no es interactivo | Formularios, menús desplegables, todo lo que reacciona a un clic |

```tsx
// app/productos/page.tsx: Server Component (no tiene "use client")
import { db } from "@/lib/db";
import { BotonAgregar } from "./boton-agregar";

export default async function Productos() {
  const productos = await db.producto.findMany(); // directo a la base, sin API pública

  return (
    <ul>
      {productos.map((p) => (
        <li key={p.id}>
          {p.nombre}: ${p.precio}
          <BotonAgregar productoId={p.id} /> {/* solo esto es de cliente */}
        </li>
      ))}
    </ul>
  );
}
```

```tsx
// app/productos/boton-agregar.tsx
"use client";

import { useState } from "react";

export function BotonAgregar({ productoId }: { productoId: number }) {
  const [agregado, setAgregado] = useState(false);
  return (
    <button onClick={() => { agregarAlCarrito(productoId); setAgregado(true); }}>
      {agregado ? "Agregado" : "Agregar"}
    </button>
  );
}
```

**Regla práctica:** empieza asumiendo que el componente es de servidor, y márcalo como cliente solo cuando necesite interactividad o hooks. Lo habitual es un Server Component que "envuelve" Client Components chicos y específicos (las "hojas" interactivas del árbol), no al revés.

### 3. El límite de `"use client"`

`"use client"` no marca solo ese archivo: marca un **límite**. Todo lo que ese archivo **importa** pasa a ser también código de cliente y viaja en el bundle.

```text
page.tsx (servidor)
 └── Filtros.tsx ("use client")      ← límite
      ├── Selector.tsx               ← cliente, aunque no tenga "use client"
      └── formatear-markdown.ts      ← cliente: ¡viaja al navegador!
```

Por eso poner `"use client"` en la raíz, como en el problema, convierte **toda** la aplicación en una SPA.

Dos reglas que se derivan:

| Situación | ¿Se puede? |
| --- | --- |
| Un Server Component **importa** un Client Component | Sí |
| Un Client Component **importa** un Server Component | No: se convierte en Client Component |
| Un Client Component **recibe** un Server Component como `children` | Sí: el servidor lo renderiza y lo pasa ya resuelto |

```tsx
// Client Component que envuelve contenido de servidor
"use client";
export function Pestanas({ children }: { children: React.ReactNode }) {
  const [abierta, setAbierta] = useState(0);
  return <div>{/* botones de pestañas */}{children}</div>;
}

// Server Component que lo usa
<Pestanas>
  <DetalleProducto id={7} /> {/* Server Component, se renderiza en el servidor */}
</Pestanas>
```

### 4. Las props que cruzan el límite tienen que ser serializables

Lo que un Server Component pasa a un Client Component viaja por la red, así que tiene que poder convertirse a texto.

Si lo usas mal:

```tsx
// Server Component
<BotonAgregar onAgregar={() => console.log("agregado")} />
```

```text
Error: Functions cannot be passed directly to Client Components unless you
explicitly expose it by marking it with "use server".
```

| Se puede pasar | No se puede pasar |
| --- | --- |
| Strings, números, booleanos, `null` | Funciones (salvo Server Actions marcadas con `"use server"`) |
| Arrays y objetos planos | Instancias de clases con métodos |
| `Date`, `Map`, `Set` | Símbolos, objetos con referencias circulares |
| JSX de Server Components (como `children`) | |

La función tiene que definirse dentro del Client Component, o ser una **Server Action** (una función con `"use server"` que se ejecuta en el servidor cuando el cliente la llama).

### 5. Hidratación: dónde puede romperse

**Hidratar** es el proceso por el que React toma el HTML que llegó del servidor y le conecta el JavaScript en el navegador, para que sea interactivo sin volver a pintar todo desde cero.

Un **hydration mismatch** ocurre cuando el HTML del servidor no coincide con lo que React genera en el cliente. Causas típicas:

| Causa | Por qué difiere |
| --- | --- |
| `Date.now()`, `new Date()` o `Math.random()` en el render | El servidor y el cliente se ejecutan en momentos distintos |
| Leer `window` o `localStorage` en el render | En el servidor no existen |
| Formatear fechas o números según el idioma del navegador | El servidor y el navegador pueden tener idiomas o zonas horarias distintos |
| HTML inválido (un `<div>` dentro de un `<p>`) | El navegador lo "corrige" y el árbol cambia |
| Extensiones del navegador que modifican el DOM | Agregan atributos o elementos antes de la hidratación |

React lo informa en la consola y, en el peor caso, vuelve a renderizar desde el cliente esa parte del árbol, perdiendo la ventaja del SSR.

La corrección habitual es mover lo que depende del navegador a un `useEffect`, que solo se ejecuta en el cliente después de hidratar:

```tsx
"use client";
export function HoraActual() {
  const [hora, setHora] = useState<string | null>(null);
  useEffect(() => setHora(new Date().toLocaleTimeString()), []);
  return <p>Actualizado: {hora ?? "—"}</p>;
}
```

Esto se conecta con la "hidratación limpia" de [la lección anterior](01-estado-cliente-vs-servidor.md): son dos capas del mismo concepto. Una es sobre el **HTML** que tiene que coincidir entre servidor y cliente; la otra, sobre los **datos** que no se tienen que volver a pedir.

### 6. Custom hooks: reutilizar lógica sin duplicarla

Un custom hook es una función que empieza con `use` y permite reutilizar **lógica con estado** entre componentes, como `useDebounce`, `useLocalStorage` o `useFormValidation`.

```tsx
function useDebounce<T>(valor: T, demoraMs = 300): T {
  const [debounced, setDebounced] = useState(valor);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(valor), demoraMs);
    return () => clearTimeout(id);
  }, [valor, demoraMs]);

  return debounced;
}
```

La ventaja frente a copiar la misma lógica en cada componente es la de [DRY](../02-calidad-de-codigo-y-arquitectura/07-principios-de-simplicidad.md): un solo lugar para corregir un bug.

> **Ojo:** un custom hook comparte **lógica**, no **estado**. Si dos componentes llaman a `useCarrito()`, cada uno tiene **su propio** estado: agregar un producto en uno no lo agrega en el otro. Para compartir el mismo estado entre componentes hace falta Context, un store o, si es estado de servidor, TanStack Query.

### 7. Prop drilling: el síntoma y las dos soluciones

**Prop drilling** es pasar una prop por 4 o 5 componentes intermedios que no la usan, solo para que llegue al que sí. Es un code smell de acoplamiento: cada intermedio depende de un dato que no le importa, y cambiar su forma obliga a tocarlos a todos.

Hay dos soluciones, y no son intercambiables:

**Composición:** en lugar de pasar el dato hacia abajo, pasas el componente ya armado como `children` o como prop. El intermedio deja de necesitar el dato.

```tsx
// Antes: Encabezado y Menu reciben "usuario" sin usarlo
<Encabezado usuario={usuario}><Menu usuario={usuario}>...</Menu></Encabezado>

// Después: el avatar se arma arriba y se pasa ya listo
<Encabezado acciones={<AvatarUsuario usuario={usuario} />} />
```

**Context:** cuando el dato es realmente "global" para una sección del árbol (tema, usuario autenticado, idioma), Context lo expone a quien lo necesite sin pasarlo por los intermedios.

```tsx
"use client";
const UsuarioContext = createContext<Usuario | null>(null);

export function UsuarioProvider({ usuario, children }: { usuario: Usuario | null; children: React.ReactNode }) {
  return <UsuarioContext.Provider value={usuario}>{children}</UsuarioContext.Provider>;
}

export const useUsuario = () => useContext(UsuarioContext);
```

| | Composición | Context |
| --- | --- | --- |
| Cuándo | El dato lo usa un solo componente, unos niveles más abajo | Muchos componentes en distintas partes del árbol usan el mismo dato |
| Costo | Ninguno | Cada cambio del valor vuelve a renderizar a **todos** los consumidores |
| Funciona en Server Components | Sí | No: Context solo existe en Client Components |

> **Ojo:** usar Context para todo termina siendo tan problemático como el prop drilling, porque cualquier cambio en el valor vuelve a renderizar a todos los consumidores. Prueba primero la composición; usa Context para datos que cambian poco y que usan muchos.

-----

## Ejemplo completo

El catálogo del problema, con arquitectura híbrida.

`app/layout.tsx` (servidor: lee el usuario una vez y lo expone con Context):

```tsx
import { obtenerUsuarioActual } from "@/lib/auth";
import { UsuarioProvider } from "@/components/usuario-provider";
import { Encabezado } from "@/components/encabezado";
import { AvatarUsuario } from "@/components/avatar-usuario";

export default async function Layout({ children }: { children: React.ReactNode }) {
  const usuario = await obtenerUsuarioActual();

  return (
    <html lang="es">
      <body>
        <UsuarioProvider usuario={usuario}>
          <Encabezado acciones={<AvatarUsuario />} />
          {children}
        </UsuarioProvider>
      </body>
    </html>
  );
}
```

`app/productos/page.tsx` (servidor: datos y Markdown se resuelven sin enviar JavaScript):

```tsx
import { db } from "@/lib/db";
import { marked } from "marked"; // se queda en el servidor: no viaja al navegador
import { BotonAgregar } from "./boton-agregar";
import { HoraActual } from "./hora-actual";

export default async function Productos() {
  const productos = await db.producto.findMany({ select: { id: true, nombre: true, precio: true, descripcion: true } });

  return (
    <>
      <HoraActual />
      <ul>
        {productos.map((p) => (
          <li key={p.id}>
            <h2>{p.nombre}</h2>
            <div dangerouslySetInnerHTML={{ __html: marked.parse(p.descripcion) as string }} />
            <BotonAgregar productoId={p.id} />
          </li>
        ))}
      </ul>
    </>
  );
}
```

`components/avatar-usuario.tsx` (cliente: lee el usuario del Context, sin prop drilling):

```tsx
"use client";
import { useUsuario } from "./usuario-provider";

export function AvatarUsuario() {
  const usuario = useUsuario();
  return usuario ? <img src={usuario.avatarUrl} alt={usuario.nombre} /> : <a href="/login">Ingresar</a>;
}
```

Qué observar:

* El HTML llega con los productos: no hay pantalla en blanco y los buscadores indexan el contenido en la primera pasada.
* La librería de Markdown se ejecuta en el servidor: no está en el bundle.
* El JavaScript del navegador se reduce a `BotonAgregar`, `HoraActual`, `AvatarUsuario` y el provider.
* `usuario` ya no atraviesa `Encabezado` ni `Menu`.
* La hora se calcula en un `useEffect`: no hay hydration mismatch.
* Las descripciones vienen de la base de datos propia; si vinieran de usuarios, habría que sanitizar el HTML antes de usar `dangerouslySetInnerHTML`.

-----

## Errores comunes

**1. `"use client"` en la raíz o en layouts.**
Qué pasa: toda la aplicación vuelve a ser una SPA, con bundle pesado y sin datos en el HTML.
Por qué: un hook en un componente de arriba "obliga" a marcarlo, y arrastra a todos sus imports.
Arreglo: extrae la parte interactiva a un componente hoja y deja el resto en el servidor.

**2. Pasar funciones de un Server Component a un Client Component.**
Qué pasa: `Functions cannot be passed directly to Client Components`.
Por qué: las props cruzan la red y las funciones no son serializables.
Arreglo: define la función dentro del Client Component, o usa una Server Action.

**3. Leer `window` o `localStorage` durante el render.**
Qué pasa: `window is not defined` en el servidor, o un hydration mismatch.
Por qué: el componente también se prerenderiza en el servidor, donde esas APIs no existen.
Arreglo: lee esos valores en un `useEffect`.

**4. Creer que un custom hook comparte estado.**
Qué pasa: dos componentes usan `useCarrito()` y cada uno muestra un carrito distinto.
Por qué: cada llamada a un hook crea su propio estado.
Arreglo: Context, un store, o TanStack Query si el carrito vive en el servidor.

**5. Context para todo.**
Qué pasa: escribir en un campo de texto vuelve a renderizar media aplicación.
Por qué: un valor de Context que cambia seguido vuelve a renderizar a todos sus consumidores.
Arreglo: composición primero; Context para datos estables; contextos separados para datos que cambian a ritmos distintos.

**6. Exponer secretos en un Client Component.**
Qué pasa: una API key aparece en el JavaScript que descarga cualquiera.
Por qué: el código de un Client Component viaja al navegador, y las variables `NEXT_PUBLIC_*` se incrustan en él.
Arreglo: los secretos solo se usan en Server Components, Route Handlers o Server Actions. El paquete `server-only` hace que el build falle si un módulo de servidor se importa desde el cliente.

-----

## Evolución y versiones

* **2010-2015:** las SPA se popularizan con AngularJS, Backbone y React.
* **2016:** Next.js nace con SSR por páginas (Pages Router, `getServerSideProps`).
* **2020:** el equipo de React presenta los Server Components como experimento.
* **2023:** Next.js 13.4 estabiliza el App Router, con Server Components por defecto.
* **2024:** React 19 estabiliza Server Components, Server Actions y mejora los mensajes de hydration mismatch.

Si encuentras `getServerSideProps` o `getStaticProps` en un tutorial, es el Pages Router: sigue funcionando, pero el App Router es el modelo actual.

-----

## Cuándo sí y cuándo no

**Usa un Server Component cuando:** el componente muestra datos o contenido sin interactividad, necesita acceder a la base de datos o a secretos, o usa librerías pesadas que no hace falta enviar al navegador.

**Usa un Client Component cuando:** necesita estado, efectos, eventos o APIs del navegador.

**No uses el modelo híbrido cuando:** es una aplicación muy interactiva detrás de un login (un editor, un panel en tiempo real), donde casi todo es de cliente y el SEO no importa. Ahí una SPA (por ejemplo, con Vite) puede ser más simple.

-----

## Resumen en 5 líneas

1. Los Server Components corren solo en el servidor y no envían JavaScript; los Client Components se hidratan y permiten interactividad.
2. Empieza en el servidor y baja al cliente solo las hojas interactivas; `"use client"` arrastra todo lo que importa.
3. Las props que cruzan al cliente tienen que ser serializables: nada de funciones (salvo Server Actions).
4. El hydration mismatch viene de render no determinista o de APIs del navegador: muévelos a un `useEffect`.
5. Custom hooks comparten lógica, no estado; contra el prop drilling, composición primero y Context para datos globales.

-----

## Para profundizar

<details>
<summary>Streaming y Suspense</summary>

Un Server Component lento (por ejemplo, unas recomendaciones que tardan 2 segundos) no tiene que bloquear toda la página. Con `Suspense`, el servidor envía primero el resto del HTML y después, en el mismo flujo de respuesta, la parte lenta cuando está lista:

```tsx
<Suspense fallback={<SkeletonRecomendaciones />}>
  <Recomendaciones /> {/* async Server Component */}
</Suspense>
```

El usuario ve la página al instante y el skeleton se reemplaza solo. En Next.js, un archivo `loading.tsx` en una carpeta de ruta hace lo mismo para toda la página.

</details>

<details>
<summary>Server Actions: mutaciones sin escribir una API</summary>

Una Server Action es una función que se ejecuta en el servidor y se puede llamar desde un formulario o un Client Component:

```tsx
// app/productos/acciones.ts
"use server";
export async function agregarAlCarrito(productoId: number) {
  const usuario = await obtenerUsuarioActual();
  if (!usuario) throw new Error("No autenticado");
  await db.carrito.create({ data: { usuarioId: usuario.id, productoId } });
  revalidatePath("/carrito");
}
```

Por debajo, Next.js crea un endpoint HTTP. Eso significa que cualquiera puede llamarlo: hay que validar los datos y verificar la autorización dentro de la acción, igual que en una API.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Los Server Components se ejecutan en el servidor y no envían JavaScript al navegador; sirven para mostrar datos. Los Client Components llevan `"use client"` y se usan cuando hace falta estado o eventos, como un botón o un formulario.

### Respuesta ampliada (semi-senior)

Uso el modelo híbrido de Next.js empezando en el servidor: los datos y el contenido se resuelven en Server Components, con acceso directo a la base y sin enviar JavaScript, y bajo al cliente solo las hojas interactivas. Sé que `"use client"` marca un límite que arrastra todo lo que importa, que las props que lo cruzan tienen que ser serializables y que un Client Component puede recibir Server Components como `children`. Evito los hydration mismatch moviendo a `useEffect` lo que depende del navegador o del momento del render. Contra el prop drilling uso composición primero y Context para datos globales y estables, sabiendo que cada cambio de Context vuelve a renderizar a todos sus consumidores.

### Preguntas frecuentes de seguimiento

**1. ¿Un Client Component se ejecuta solo en el navegador?**
No. También se prerenderiza en el servidor para generar el HTML inicial, y después se hidrata en el navegador. Por eso no puede leer `window` durante el render.

**2. ¿Qué es un hydration mismatch y cómo lo evitas?**
Que el HTML del servidor no coincide con el render del cliente. Lo evito con un render determinista: nada de `Date.now()`, `Math.random()` ni APIs del navegador fuera de `useEffect`.

**3. ¿Cómo resolverías el prop drilling?**
Primero con composición, pasando componentes ya armados como `children` o props. Si el dato lo usan muchos componentes en distintas partes del árbol, con Context.

-----

## Práctica

**Ejercicio 1.** Decide si cada componente debería ser de servidor o de cliente.

1. El pie de página con los links legales.
2. Un buscador con sugerencias mientras escribes.
3. La lista de los 20 artículos más recientes del blog.
4. El botón "Me gusta" de cada artículo.
5. Un gráfico que usa una librería que lee `window.innerWidth`.
6. El detalle de un producto que lee el precio de la base de datos.

<details>
<summary>Solución</summary>

1. **Servidor:** contenido estático.
2. **Cliente:** estado del texto y eventos de teclado.
3. **Servidor:** datos sin interactividad.
4. **Cliente:** estado y `onClick`. Es la hoja interactiva dentro del artículo de servidor.
5. **Cliente**, y si la librería falla al prerenderizar en el servidor, cárgala con `next/dynamic` y `ssr: false` desde un Client Component.
6. **Servidor:** acceso directo a la base de datos.

</details>

**Ejercicio 2.** ¿Qué problemas tiene este componente? Corrígelo.

```tsx
"use client";

export default function Perfil({ usuario }: { usuario: Usuario }) {
  const tema = localStorage.getItem("tema") ?? "claro";
  return (
    <div className={tema}>
      <p>Hola, {usuario.nombre}. Hoy es {new Date().toLocaleDateString()}.</p>
    </div>
  );
}
```

<details>
<summary>Solución</summary>

* `localStorage` en el render: falla en el servidor (`localStorage is not defined`).
* `new Date().toLocaleDateString()`: puede diferir entre servidor y cliente (zona horaria e idioma): hydration mismatch.
* Todo el componente es de cliente aunque el saludo no es interactivo.

```tsx
// Perfil.tsx (servidor)
export default function Perfil({ usuario }: { usuario: Usuario }) {
  return (
    <ContenedorConTema>
      <p>Hola, {usuario.nombre}. Hoy es <FechaLocal />.</p>
    </ContenedorConTema>
  );
}

// ContenedorConTema.tsx
"use client";
export function ContenedorConTema({ children }: { children: React.ReactNode }) {
  const [tema, setTema] = useState("claro");
  useEffect(() => setTema(localStorage.getItem("tema") ?? "claro"), []);
  return <div className={tema}>{children}</div>;
}

// FechaLocal.tsx
"use client";
export function FechaLocal() {
  const [fecha, setFecha] = useState<string | null>(null);
  useEffect(() => setFecha(new Date().toLocaleDateString()), []);
  return <>{fecha ?? ""}</>;
}
```

El saludo queda en el servidor; solo el tema y la fecha local se resuelven en el cliente, después de hidratar.

</details>

-----

## Siguiente lección

[Rendimiento y profiling](03-rendimiento-y-profiling.md)
