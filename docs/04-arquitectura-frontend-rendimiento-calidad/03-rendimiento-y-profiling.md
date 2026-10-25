# Rendimiento y profiling de cliente

## En una frase

El rendimiento en el frontend se **mide antes de optimizarse**: el Profiler de React muestra qué se vuelve a renderizar sin necesidad, los Core Web Vitals (LCP, INP, CLS) miden lo que el usuario percibe, el code splitting achica lo que se descarga, y la pestaña Memory de Chrome encuentra lo que nunca se libera.

-----

## Antes de empezar

Conviene que ya sepas:

* Server y Client Components, y qué es el bundle: [Server y Client Components](02-server-client-components.md).
* Que optimizar sin medir no funciona: [Depuración avanzada](../01-autonomia-resolucion-problemas/02-depuracion-avanzada.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Render (re-render):** cuando React vuelve a ejecutar un componente para calcular qué mostrar.
* **Memoizar:** guardar el resultado de un cálculo o una referencia para no recalcularla si sus entradas no cambiaron.
* **Igualdad referencial:** dos objetos son "iguales" para React solo si son **el mismo** objeto en memoria, no si tienen el mismo contenido.
* **Flamegraph:** gráfico que muestra qué componentes se renderizaron y cuánto tardó cada uno.
* **Core Web Vitals:** las tres métricas de Google para la experiencia percibida: LCP, INP y CLS.
* **Code splitting:** dividir el bundle en partes que se descargan solo cuando hacen falta.
* **Lazy loading:** cargar un recurso (componente, imagen) recién cuando se necesita.
* **Memory leak (fuga de memoria):** memoria que la aplicación ya no usa pero que no se libera.
* **Heap snapshot:** fotografía de todos los objetos en memoria en un instante.
* **Condición de carrera:** el resultado depende del orden en que terminan operaciones asíncronas.

-----

## El problema

Un dashboard de ventas "anda lento". El equipo envuelve todos los componentes en `React.memo` y todas las funciones en `useCallback`. Sigue lento.

Lo que pasa de verdad:

```tsx
function Dashboard() {
  const [busqueda, setBusqueda] = useState("");
  const ventas = useVentas(); // 5000 filas

  return (
    <>
      <input value={busqueda} onChange={(e) => setBusqueda(e.target.value)} />
      <TablaVentas
        filas={ventas.filter((v) => v.cliente.includes(busqueda))}
        opciones={{ ordenable: true }}           // objeto nuevo en cada render
      />
      <img src="/banner.jpg" />                  {/* sin dimensiones */}
      <GraficoAnual datos={ventas} />            {/* 300 KB de librería de gráficos */}
    </>
  );
}

function GraficoAnual({ datos }: { datos: Venta[] }) {
  useEffect(() => {
    window.addEventListener("resize", redibujar); // nunca se quita
  }, []);
  // ...
}
```

Las consecuencias:

* cada tecla en el buscador vuelve a renderizar las 5000 filas; el `React.memo` de `TablaVentas` no sirve porque `opciones` es un objeto nuevo en cada render;
* el banner aparece tarde y empuja la tabla hacia abajo mientras el usuario intenta hacer clic;
* los 300 KB del gráfico se descargan aunque esté debajo, fuera de la pantalla;
* después de navegar 20 veces entre pantallas, la pestaña ocupa 900 MB.

-----

## Cómo funciona

### 1. Cuándo vuelve a renderizar React

React vuelve a renderizar un componente cuando:

* cambia su estado;
* cambia un Context que consume;
* se vuelve a renderizar su componente padre (aunque sus props no hayan cambiado).

El problema no es que renderice, sino que lo haga **sin necesidad**: un padre que se renderiza arrastra a todos sus hijos aunque ninguno haya cambiado.

> **Render no es lo mismo que actualizar el DOM.** Un render es ejecutar la función del componente y comparar el resultado. Si nada cambió, React no toca el DOM. Muchos renders son baratos; el problema aparece con componentes costosos (listas grandes, cálculos pesados) o con renders en cadena.

### 2. Medir primero: el Profiler de React

El **Profiler** de React DevTools (extensión del navegador) graba una interacción y muestra un flamegraph con qué componentes se renderizaron, cuánto tardó cada uno y **por qué** se renderizó.

```text
1. React DevTools → pestaña Profiler
2. ⚙ → activar "Record why each component rendered while profiling"
3. ● Grabar → hacer la interacción lenta (escribir en el buscador) → ■ Detener
4. Buscar las barras más anchas y amarillas: son los renders más costosos
```

Sin esta herramienta, "optimizar el rendimiento" es adivinar. En el problema, el Profiler habría mostrado que `TablaVentas` tarda 180 ms en cada tecla, y que se renderiza porque "cambió la prop `opciones`".

### 3. `React.memo`, `useMemo` y `useCallback`

| Herramienta | Qué memoiza | Para qué |
| --- | --- | --- |
| `React.memo(Componente)` | El componente | No volver a renderizarlo si sus props no cambiaron (comparación superficial) |
| `useMemo(() => calculo, [deps])` | El **resultado** de un cálculo | No recalcular algo costoso, o mantener la misma referencia de un objeto o array |
| `useCallback(fn, [deps])` | La **referencia** de una función | Pasar la misma función a un hijo memoizado |

`React.memo` compara las props por **igualdad referencial**. Para React, `{ ordenable: true }` creado en dos renders son dos objetos distintos.

Si lo usas mal:

```tsx
const TablaVentas = React.memo(Tabla);

// en el padre:
<TablaVentas filas={filas} opciones={{ ordenable: true }} onSeleccionar={(id) => abrir(id)} />
```

`opciones` y `onSeleccionar` son referencias nuevas en cada render, así que `React.memo` compara, ve props "distintas" y renderiza igual. Pagas el costo de comparar **y** el de renderizar. Para que funcione, el objeto y la función también tienen que tener referencias estables:

```tsx
const opciones = useMemo(() => ({ ordenable: true }), []); // o una constante fuera del componente
const onSeleccionar = useCallback((id: number) => abrir(id), [abrir]);
```

**La trampa real:** `useMemo` y `useCallback` no son gratis: tienen un costo de memoria y de comparación de dependencias. Usados "por si acaso" en todos lados, muchas veces cuestan más que recalcular. **Regla: mide con el Profiler y memoiza donde el flamegraph muestre un problema real.**

> **React Compiler:** desde su versión estable (1.0, octubre de 2025), el React Compiler analiza el código en el build y agrega la memoización automáticamente donde hace falta. En un proyecto que lo usa, la mayoría de los `useMemo`, `useCallback` y `React.memo` manuales ya no son necesarios. Entender el problema de la igualdad referencial sigue siendo clave para leer código existente y para saber qué hace el compilador.

### 4. Antes de memoizar: mover el estado hacia abajo

Muchas veces la mejor optimización no es memoizar, sino que el estado viva **más cerca** de quien lo usa, para que su cambio no arrastre a componentes que no lo necesitan.

```tsx
// Antes: escribir en el buscador re-renderiza GraficoAnual
function Dashboard() {
  const [busqueda, setBusqueda] = useState("");
  return (<><Buscador valor={busqueda} onCambio={setBusqueda} /><GraficoAnual /></>);
}

// Después: el estado del buscador vive en su propio componente
function Dashboard() {
  return (<><SeccionVentas /><GraficoAnual /></>);
}

function SeccionVentas() {
  const [busqueda, setBusqueda] = useState("");
  return (<><Buscador valor={busqueda} onCambio={setBusqueda} /><TablaVentas busqueda={busqueda} /></>);
}
```

Ahora escribir en el buscador no toca `GraficoAnual`, sin ningún `memo`.

### 5. Core Web Vitals

Google mide el rendimiento **percibido** con tres métricas centrales:

| Métrica | Mide | Bueno | Causa típica de un mal valor |
| --- | --- | --- | --- |
| **LCP** (*Largest Contentful Paint*) | Cuánto tarda en pintarse el elemento más grande visible (la imagen o el bloque de texto principal) | ≤ 2,5 s | Imagen principal pesada o que se descubre tarde, servidor lento, JavaScript que bloquea |
| **INP** (*Interaction to Next Paint*) | Cuánto tarda la interfaz en responder visualmente después de un clic, toque o tecla | ≤ 200 ms | Tareas de JavaScript largas en el hilo principal, renders costosos |
| **CLS** (*Cumulative Layout Shift*) | Cuánto "salta" el contenido mientras carga | ≤ 0,1 | Imágenes sin dimensiones, anuncios o banners que se insertan arriba, fuentes que cambian de tamaño |

Google evalúa cada métrica en el **percentil 75** de las visitas reales: para aprobar, al menos el 75 % de los usuarios tiene que tener una buena experiencia.

> **Si estudiaste con material anterior a 2024:** hasta marzo de 2024, la tercera métrica era **FID** (*First Input Delay*), que solo medía el retraso del **primer** input. INP la reemplazó porque mide la capacidad de respuesta durante **toda** la sesión. Es más completa y más difícil de "maquillar" optimizando solo el primer evento.

**Datos de laboratorio frente a datos de campo:**

| | Laboratorio | Campo |
| --- | --- | --- |
| Qué es | Una medición simulada en tu máquina | Lo que miden los navegadores de usuarios reales |
| Herramientas | Lighthouse, pestaña Performance de Chrome | Chrome UX Report, PageSpeed Insights (sección de campo), librería `web-vitals` |
| Sirve para | Diagnosticar y comparar cambios | Saber cómo lo viven de verdad los usuarios |

Un 100 en Lighthouse en tu notebook con fibra óptica no dice cómo lo vive alguien con un teléfono de gama baja y 4G.

### 6. Lazy loading y code splitting

Cargar todo el JavaScript de una vez infla el bundle inicial. **Code splitting** lo divide en partes que se descargan solo cuando hacen falta.

| Técnica | Cómo | Cuándo |
| --- | --- | --- |
| Por ruta | Automático en Next.js: cada página es su propio chunk | Siempre: el usuario no descarga rutas que no visita |
| `React.lazy` + `Suspense` | `const Modal = lazy(() => import("./Modal"))` | Componentes pesados que no siempre se usan: un modal, un editor |
| `next/dynamic` | `dynamic(() => import("./Grafico"), { ssr: false })` | Lo mismo en Next.js, con la opción de no prerenderizar en el servidor un componente que depende del navegador |
| Imágenes diferidas | `<img loading="lazy">` o `next/image` | Imágenes fuera de la primera pantalla |

> **Ojo con `ssr: false` en el App Router:** solo se puede usar dentro de un **Client Component**. Si lo pones en un Server Component, Next.js da error. La solución es un pequeño Client Component que haga el `dynamic(...)`.

> **Ojo con la imagen del LCP:** **nunca** le pongas `loading="lazy"` a la imagen principal de la página. Es justamente la que mide el LCP, y diferirla lo empeora. A esa imagen, al contrario, conviene darle prioridad (`priority` en `next/image`, o `fetchpriority="high"`).

### 7. Memory leaks en el cliente

Un memory leak en el frontend suele venir de una de estas causas:

**1. Suscripciones sin limpiar.** Un `useEffect` que agrega un listener o una suscripción y no devuelve una función de limpieza:

```tsx
// Mal: cada vez que se monta, agrega otro listener que nunca se quita
useEffect(() => {
  window.addEventListener("resize", redibujar);
}, []);

// Bien: la función de limpieza se ejecuta al desmontar
useEffect(() => {
  window.addEventListener("resize", redibujar);
  return () => window.removeEventListener("resize", redibujar);
}, []);
```

Lo mismo aplica a `setInterval` (`clearInterval`), WebSockets (`close()`) y suscripciones a librerías externas.

**2. Referencias que sobreviven al componente.** Un callback guardado en algo de vida larga (un objeto global, una caché del módulo, una librería externa) que sigue referenciando el estado de un componente ya desmontado: el componente nunca se puede liberar.

**3. Cachés que solo crecen.** Un `Map` a nivel de módulo donde se guarda todo lo que se pide y nunca se borra nada.

> **Ojo: el "setState después de desmontar" no es un memory leak.** React 17 y anteriores mostraban la advertencia *"Can't perform a React state update on an unmounted component... indicates a memory leak"*. React 18 **la quitó**, porque en la gran mayoría de los casos no había ninguna fuga. El problema real de una petición que responde tarde es otro: una **condición de carrera**. Si el usuario cambia rápido de pedido 1 a pedido 2, la respuesta de 1 puede llegar **después** y mostrar datos incorrectos. Se resuelve cancelando la petición anterior:
>
> ```tsx
> useEffect(() => {
>   const controller = new AbortController();
>   fetch(`/api/orders/${id}`, { signal: controller.signal })
>     .then((r) => r.json())
>     .then(setPedido)
>     .catch((e) => { if (e.name !== "AbortError") setError(e); });
>   return () => controller.abort();
> }, [id]);
> ```
>
> (TanStack Query resuelve esto solo, porque cada key tiene su propia entrada de caché).

**Cómo detectarlos:** la pestaña **Memory** de Chrome DevTools.

```text
1. Memory → Heap snapshot → tomar una fotografía (línea base)
2. Repetir 10 veces la interacción sospechosa (abrir y cerrar el modal, navegar ida y vuelta)
3. Forzar la recolección de basura (ícono del tacho) y tomar otra fotografía
4. Comparar: vista "Comparison", ordenar por "# Delta"
5. Si crece una clase de objetos (componentes, listeners "Detached") en cada repetición, hay una fuga
```

Si el heap crece de forma sostenida y no baja después de la recolección de basura, hay un leak. Es un uso distinto de DevTools al de inspeccionar CSS en la pestaña Elements: son la misma herramienta, pero resuelven problemas distintos.

-----

## Ejemplo completo

El dashboard del problema, medido y corregido.

**1. Medir.** El Profiler muestra que `TablaVentas` tarda 180 ms por tecla. Lighthouse marca un CLS de 0,32 y un bundle inicial de 520 KB. Después de 20 navegaciones, el heap pasa de 60 MB a 900 MB, con cientos de listeners `resize`.

**2. Corregir solo lo medido:**

```tsx
"use client"; // usa hooks y dynamic con ssr: false

import dynamic from "next/dynamic";
import Image from "next/image";
import { useDeferredValue, useMemo, useState } from "react";

const GraficoAnual = dynamic(() => import("./grafico-anual"), {
  ssr: false,
  loading: () => <div style={{ height: 320 }} />, // reserva el espacio: sin CLS
});

const OPCIONES_TABLA = { ordenable: true }; // referencia estable, fuera del componente

export function Dashboard() {
  const ventas = useVentas();

  return (
    <>
      <SeccionVentas ventas={ventas} />
      <Image src="/banner.jpg" alt="Campaña de verano" width={1200} height={300} priority />
      <GraficoAnual datos={ventas} />
    </>
  );
}

function SeccionVentas({ ventas }: { ventas: Venta[] }) {
  const [busqueda, setBusqueda] = useState("");
  const busquedaDiferida = useDeferredValue(busqueda);

  const filtradas = useMemo(
    () => ventas.filter((v) => v.cliente.toLowerCase().includes(busquedaDiferida.toLowerCase())),
    [ventas, busquedaDiferida]
  );

  return (
    <>
      <input aria-label="Buscar cliente" value={busqueda} onChange={(e) => setBusqueda(e.target.value)} />
      <TablaVentas filas={filtradas} opciones={OPCIONES_TABLA} />
    </>
  );
}
```

`grafico-anual.tsx`:

```tsx
"use client";

export default function GraficoAnual({ datos }: { datos: Venta[] }) {
  useEffect(() => {
    const redibujar = () => { /* ... */ };
    window.addEventListener("resize", redibujar);
    return () => window.removeEventListener("resize", redibujar);
  }, []);
  // ...
}
```

**3. Volver a medir.**

| Métrica | Antes | Después |
| --- | --- | --- |
| Render de `TablaVentas` por tecla | 180 ms | No bloquea la escritura (`useDeferredValue`) |
| CLS | 0,32 | 0,02 |
| Bundle inicial | 520 KB | 220 KB |
| Heap tras 20 navegaciones | 900 MB | 65 MB, estable |

Qué observar:

* Se memoizó **una** cosa (`filtradas`) y se estabilizó **una** referencia (`OPCIONES_TABLA`), no todo el árbol.
* El estado de la búsqueda bajó a `SeccionVentas`: escribir ya no toca el gráfico ni el banner.
* `useDeferredValue` deja que el input responda al instante y que la tabla se actualice cuando React tenga tiempo: mejora el INP.
* El banner tiene dimensiones (sin CLS) y prioridad (mejor LCP); el gráfico se descarga aparte y reserva su espacio.
* Cada cambio se validó con una nueva medición, no con intuición.

-----

## Errores comunes

**1. Memoizar todo "por si acaso".**
Qué pasa: código más difícil de leer y la aplicación igual de lenta.
Por qué: se optimiza sin medir, y la memoización tiene costo.
Arreglo: Profiler primero; memoiza solo donde el flamegraph muestra el problema.

**2. `React.memo` con props que cambian de referencia.**
Qué pasa: el componente se renderiza igual, y además se paga la comparación.
Por qué: objetos, arrays o funciones creados en cada render del padre.
Arreglo: constantes fuera del componente, `useMemo` o `useCallback` para esas props, o el React Compiler.

**3. `useEffect` sin función de limpieza.**
Qué pasa: listeners e intervalos duplicados, memoria que crece en cada navegación.
Por qué: se agrega la suscripción y se olvida quitarla al desmontar.
Arreglo: todo `useEffect` que suscribe devuelve una función que desuscribe.

**4. Imágenes sin dimensiones.**
Qué pasa: el contenido salta cuando cargan; el usuario hace clic en el lugar equivocado.
Por qué: el navegador no sabe cuánto espacio reservar.
Arreglo: `width` y `height` (o `aspect-ratio` en CSS), o `next/image`.

**5. `loading="lazy"` en la imagen principal.**
Qué pasa: el LCP empeora.
Por qué: se difiere justo el elemento que la métrica está esperando.
Arreglo: lazy solo para imágenes fuera de la primera pantalla; la principal, con prioridad.

**6. Optimizar con datos de laboratorio solamente.**
Qué pasa: 100 en Lighthouse y usuarios que se quejan de lentitud.
Por qué: tu máquina y tu conexión no son las de tus usuarios.
Arreglo: mide en campo (librería `web-vitals`, Chrome UX Report) y prueba con la CPU y la red limitadas en DevTools.

-----

## Evolución y versiones

* **2018:** React 16.6 agrega `React.memo` y `React.lazy`; React 16.8, los hooks (`useMemo`, `useCallback`).
* **2020:** Google presenta los Core Web Vitals (LCP, FID, CLS).
* **2022:** React 18 agrega `useDeferredValue` y `useTransition`, y quita la advertencia de "setState en un componente desmontado".
* **Marzo de 2024:** INP reemplaza a FID como Core Web Vital.
* **Octubre de 2025:** React Compiler 1.0: memoización automática en el build.

-----

## Cuándo sí y cuándo no

**Optimiza cuando:** una medición (Profiler, Web Vitals de campo, heap) muestra un problema concreto que afecta a los usuarios.

**Usa `useMemo` o `useCallback` cuando:** el cálculo es costoso de verdad, o la referencia se pasa a un hijo memoizado o a las dependencias de un efecto.

**No optimices cuando:** no mediste, el componente es chico y barato, o la "mejora" hace el código más difícil de mantener sin un beneficio medible.

-----

## Resumen en 5 líneas

1. Mide antes de optimizar: Profiler para renders, Web Vitals de campo para la experiencia, heap snapshots para la memoria.
2. `React.memo` compara por referencia: objetos y funciones nuevos en cada render lo anulan; antes de memoizar, prueba bajar el estado.
3. Core Web Vitals: LCP ≤ 2,5 s, INP ≤ 200 ms (reemplazó a FID en 2024), CLS ≤ 0,1, en el percentil 75.
4. Code splitting por ruta y lazy loading para lo pesado y lo que está fuera de pantalla, nunca para la imagen principal.
5. Todo `useEffect` que suscribe limpia; las respuestas tardías son condiciones de carrera, que se resuelven con `AbortController`.

-----

## Para profundizar

<details>
<summary>Listas muy largas: virtualización</summary>

Si una tabla tiene 10 000 filas, aunque memoices todo, el DOM tiene 10 000 filas. La **virtualización** renderiza solo las filas visibles (más unas pocas de margen) y las va reemplazando al hacer scroll. Librerías como TanStack Virtual o `react-window` lo resuelven. Pasa de miles de nodos del DOM a unas decenas, y suele ser la mejora más grande en pantallas con listas grandes.

</details>

<details>
<summary>INP: tareas largas y ceder el hilo principal</summary>

El navegador ejecuta JavaScript, calcula estilos y pinta en un solo hilo. Una tarea de más de 50 ms (una "tarea larga") bloquea la respuesta a los clics. Para mejorar el INP:

* `useTransition` y `useDeferredValue` para marcar actualizaciones como no urgentes;
* dividir cálculos grandes en partes y ceder el hilo entre ellas (`await scheduler.yield()` en navegadores que lo soportan, o `setTimeout`);
* mover cálculos pesados a un Web Worker.

La pestaña **Performance** de Chrome marca las tareas largas con un triángulo rojo.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para mejorar el rendimiento en React uso `React.memo`, `useMemo` y `useCallback` para evitar renders innecesarios, y lazy loading para cargar componentes cuando se necesitan. Google mide el rendimiento con los Core Web Vitals: LCP, INP y CLS.

### Respuesta ampliada (semi-senior)

Primero mido: con el Profiler de React veo qué se renderiza y por qué, y con datos de campo de Web Vitals veo cómo lo viven los usuarios. Antes de memoizar intento bajar el estado al componente que lo usa; si memoizo, cuido la igualdad referencial de las props, porque un objeto nuevo en cada render anula el `React.memo`, y sé que el React Compiler ya automatiza gran parte de eso. Para el LCP priorizo la imagen principal y reduzco el JavaScript inicial con code splitting; para el INP divido las tareas largas y uso `useDeferredValue`; para el CLS reservo el espacio de imágenes y contenido diferido. Y para la memoria, todo efecto que suscribe limpia, y verifico con heap snapshots comparando antes y después de repetir una interacción.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué `React.memo` a veces no evita el render?**
Porque compara las props por referencia: si el padre crea un objeto, array o función nuevos en cada render, la prop "cambió" aunque tenga el mismo contenido.

**2. ¿Qué reemplazó a FID y por qué?**
INP, en marzo de 2024. FID medía solo el retraso del primer input; INP mide la respuesta a las interacciones de toda la sesión.

**3. ¿Cómo encontrarías un memory leak?**
Con heap snapshots en Chrome: una línea base, repetir la interacción varias veces, forzar la recolección de basura, otra fotografía, y comparar qué tipos de objetos crecen en cada repetición.

-----

## Práctica

**Ejercicio 1.** ¿Por qué este `React.memo` no sirve? Corrígelo de dos formas distintas.

```tsx
const Lista = React.memo(function Lista({ items, estilo }: Props) { /* ... */ });

function Pagina() {
  const [contador, setContador] = useState(0);
  const items = useItems();
  return (
    <>
      <button onClick={() => setContador(contador + 1)}>{contador}</button>
      <Lista items={items} estilo={{ color: "red" }} />
    </>
  );
}
```

<details>
<summary>Solución</summary>

`estilo={{ color: "red" }}` crea un objeto nuevo en cada render de `Pagina`, así que cada clic en el botón hace que `Lista` vea una prop distinta y se vuelva a renderizar.

**Forma 1: referencia estable.**

```tsx
const ESTILO_LISTA = { color: "red" };
// ...
<Lista items={items} estilo={ESTILO_LISTA} />
```

**Forma 2: bajar el estado.** El contador no tiene nada que ver con la lista:

```tsx
function Contador() {
  const [contador, setContador] = useState(0);
  return <button onClick={() => setContador(contador + 1)}>{contador}</button>;
}

function Pagina() {
  const items = useItems();
  return (<><Contador /><Lista items={items} estilo={{ color: "red" }} /></>);
}
```

Con la forma 2, `Pagina` ya no se renderiza en cada clic, así que `Lista` tampoco, y ni siquiera hace falta el `React.memo`.

</details>

**Ejercicio 2.** ¿Qué problema tiene este efecto? Corrígelo.

```tsx
function Notificaciones() {
  const [cantidad, setCantidad] = useState(0);

  useEffect(() => {
    setInterval(async () => {
      const r = await fetch("/api/notifications/count");
      setCantidad((await r.json()).count);
    }, 5000);
  }, []);

  return <span>{cantidad}</span>;
}
```

<details>
<summary>Solución</summary>

El `setInterval` nunca se limpia. Cada vez que el componente se monta (al navegar de vuelta, o dos veces en desarrollo con `StrictMode`), se agrega **otro** intervalo, y todos siguen haciendo peticiones aunque el componente ya no esté en pantalla.

```tsx
useEffect(() => {
  const controller = new AbortController();

  const id = setInterval(async () => {
    try {
      const r = await fetch("/api/notifications/count", { signal: controller.signal });
      if (r.ok) setCantidad((await r.json()).count);
    } catch (e) {
      if ((e as Error).name !== "AbortError") console.error(e);
    }
  }, 5000);

  return () => {
    clearInterval(id);
    controller.abort();
  };
}, []);
```

Alternativa más simple con TanStack Query: `useQuery({ queryKey: ["notificaciones"], queryFn, refetchInterval: 5000 })`, que se detiene solo al desmontar.

</details>

-----

## Siguiente lección

[Estrategia de testing en UI](04-estrategia-de-testing-ui.md)
