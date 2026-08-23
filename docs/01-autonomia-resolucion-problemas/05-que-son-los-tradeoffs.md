# ¿Qué son los Trade-offs?

> Resumen basado en el artículo "Software Engineering Trade-Offs".

## Definición, en simple

Un **trade-off** (compromiso) pasa cuando, para mejorar algo, necesariamente tienes que sacrificar otra cosa. No existe la opción de "tener todo": más rápido, más simple, más flexible, más barato, todo al mismo tiempo. Eliges qué priorizar según lo que el proyecto necesita en ese momento.

Es como comprar un auto: no existe el auto que sea el más barato, el más rápido y el que gasta menos combustible, todo junto. Eliges según lo que más te importa en ese momento.

En programación es exactamente igual. Por ejemplo: cuando evalúas 2-3 alternativas antes de escalar un problema (ver [01-investigacion-y-criterio.md](./01-investigacion-y-criterio.md)), lo que en realidad estás haciendo es comparar qué gana y qué pierde cada opción. Eso es evaluar trade-offs.

## No existe el sistema perfecto

Un sistema ideal cumpliría cada requerimiento, manejaría todos los casos raros, y encima sería simple y fácil de mantener. En la vida real, eso no pasa, porque:

- Cada caso nuevo que el sistema tiene que soportar, le agrega complejidad.
- Cada librería o herramienta nueva que agregas, también.
- Si priorizas que algo sea simple de mantener, probablemente sacrificas flexibilidad para el futuro. Y viceversa.

Buena noticia: en software, a diferencia de otras decisiones de la vida, casi nada es 100% irreversible. En el peor de los casos, siempre existe la opción de reescribir una parte del sistema. Va a costar tiempo y dinero, pero es una salida real.

## Los 8 trade-offs principales

### 1. Complejidad vs. todo lo demás

Casi cualquier decisión que tomas en el código agrega complejidad. No hay vuelta que darle — el objetivo no es "cero complejidad" (imposible), sino mantenerla lo más baja posible para lo que el proyecto realmente necesita.

**Ejemplo:** tu API exporta datos en CSV. Te piden agregar Excel. Y después PDF. Y después XML. Cada formato nuevo significa más código, más tests, y más lugares donde algo se puede romper. En algún punto hay que preguntarse: ¿de verdad necesitamos los 4 formatos, o con 2 alcanza?

**Costo de la complejidad alta:** cuesta más mantener el sistema, cuesta más que alguien nuevo lo entienda (onboarding), cuesta más agregar cada cambio nuevo, y en el peor caso, el sistema se vuelve tan complicado que da miedo tocarlo.

### 2. Simplicidad vs. flexibilidad

- **Simplicidad** = código fácil de entender y mantener hoy. 

- **Flexibilidad** = código preparado para adaptarse a cambios futuros.

**Ejemplo:** estás armando un sistema de notificaciones. Hoy solo se necesita mandar emails. Puedes:
- Escribir una función simple que manda el email (simple, pero si mañana piden SMS, hay que reescribir).
- Armar una interfaz genérica `INotificationSender` con implementaciones intercambiables para email, SMS, push, Slack — "por si algún día se necesita" (flexible, pero hoy es código de más que nadie usa, y es más difícil de entender para alguien que recién llega).

| | Simplicidad | Flexibilidad |
|---|---|---|
| A favor | Se entiende rápido, hay menos cosas que puedan fallar | Es más fácil de extender cuando de verdad llega el cambio |
| En contra | Puede necesitar una reescritura si el requerimiento cambia antes de lo esperado | El código "por las dudas" infla el proyecto sin necesidad, y es más difícil de testear |

Regla de oro: **No Big Design Up Front** — no diseñes hoy para todos los escenarios futuros que te imaginas. La mitad de esos escenarios nunca va a pasar, y la otra mitad va a ser distinta a como la imaginaste.

### 3. Time-to-Market vs. deuda técnica

- **Time-to-Market** = qué tan rápido lanzas algo al mercado. 

- **Deuda técnica** = todo lo que dejas "para después" (tests, buena arquitectura, seguridad, observabilidad) para lanzar más rápido ahora.

**Ejemplo:** tu equipo tiene 2 semanas para el MVP. Puedes escribir tests completos y una arquitectura prolija (tarda más, pero es sólido), o saltarte los tests y hardcodear cosas para llegar a tiempo (rápido ahora, pero cada bug futuro te va a costar más caro arreglarlo).

| | Lanzar rápido | Invertir en calidad ahora |
|---|---|---|
| A favor | Llegas antes a los usuarios, consigues feedback real más rápido, generas ingresos antes | Menos bugs y caídas a futuro, las próximas features se construyen más rápido sobre una base sólida |
| En contra | La deuda se acumula y cuesta más arreglarla después; más riesgo de bugs y problemas de seguridad | Lanzas más tarde, un competidor te puede ganar el mercado mientras "pules" |

Ojo: deuda técnica no es solo "código feo". No tener tests, no tener logs, no tener seguridad revisada, también es deuda técnica — aunque el código se vea prolijo.

### 4. Escalado horizontal vs. vertical

**Vertical** = hacer más potente la misma máquina (más RAM, más CPU). **Horizontal** = agregar más máquinas que se reparten el trabajo.

**Ejemplo:** tu API empieza a recibir más tráfico del que soporta. Puedes subir de plan tu servidor actual (vertical, simple, pero tiene un techo — en algún punto no hay servidor más grande que comprar), o poner 3 instancias de tu API detrás de un balanceador de carga (horizontal, escala casi sin límite, pero ahora tienes que pensar en cosas como: ¿qué pasa si el usuario pega en la instancia 1 y después en la 2? ¿Comparten sesión?).

| | Vertical | Horizontal |
|---|---|---|
| A favor | Más simple de operar, no hay que pensar en coordinar varias máquinas | Escala casi sin límite, si una máquina se cae las otras siguen funcionando |
| En contra | Tiene un techo, y es un único punto de falla | Más complejo de operar, hay que diseñar pensando en que el estado esté compartido entre instancias |

Importante: cambiar de un modelo al otro más adelante no es gratis — normalmente implica repensar buena parte de la arquitectura.

### 5. Latencia vs. throughput

**Latencia** = cuánto tarda en responder UN request puntual. **Throughput** = cuántos requests en total puedes procesar por segundo.

Parece que optimizar uno mejora el otro automáticamente, pero no siempre es así. Optimizar latencia significa dedicarle recursos extra (CPU, memoria) a UN request para que vuele. Optimizar throughput significa repartir esos mismos recursos entre MUCHOS requests, priorizando el promedio general sobre la velocidad de uno solo.

**Ejemplo:** un buscador que responde en 50ms para un solo usuario está optimizado en latencia. Un sistema de procesamiento de pagos en lote (batch) que procesa 10.000 pagos por minuto, aunque cada pago individual tarde 2 segundos, está optimizado en throughput.

Términos que vas a escuchar seguido:
- **SLO** (Service Level Objective): un objetivo concreto que te pones, ej: "el 99% de los requests deben responder en menos de 200ms".
- **p99**: "percentil 99" — significa que el 99% de tus requests responden en ese tiempo o menos (el 1% restante puede ser más lento).
- **Back-pressure**: cuando al sistema le llega más trabajo del que puede procesar, y empieza a acumularse (como una cola que no para de crecer).

Recomendación práctica: si no tienes claro cuál priorizar, define primero un SLO (ej: "p99 menor a 300ms") y después optimiza throughput sin romper ese límite.

### 6. Stateful vs. stateless

**Stateful** = el servidor guarda información sobre lo que el usuario venía haciendo. **Stateless** = cada request trae toda la información que necesita, el servidor no "recuerda" nada entre uno y otro.

**Ejemplo:** un carrito de compras guardado en memoria del servidor (stateful) es rápido y simple mientras el usuario pegue siempre en la misma instancia — pero si escalas horizontalmente y el usuario cae en otra instancia, el carrito "desaparece". Guardar el carrito en una base de datos o en Redis, accesible desde cualquier instancia (stateless desde el punto de vista del servidor web), soluciona eso, a cambio de una llamada de red extra en cada request.

| | Stateful | Stateless |
|---|---|---|
| A favor | Más simple de programar, menos llamadas de red | Escala mejor, si una instancia se cae no se pierde nada |
| En contra | Difícil de escalar horizontalmente, si el servidor se cae se pierde el estado | Hay que viajar el estado por la red en cada request, más complejo de coordinar |

### 7. Síncrono (bloqueante) vs. asíncrono (no bloqueante)

**Síncrono** = tu código espera parado a que termine una operación (ej: una llamada a una API externa) antes de seguir. **Asíncrono** = tu código sigue haciendo otras cosas mientras esa operación termina en segundo plano.

**Ejemplo en C#:**

```csharp
// Síncrono (bloqueante): el hilo se queda esperando parado
var response = httpClient.GetAsync(url).Result;

// Asíncrono (no bloqueante): el hilo queda libre para atender otras cosas mientras espera
var response = await httpClient.GetAsync(url);
```

| | Síncrono | Asíncrono |
|---|---|---|
| A favor | Más fácil de leer y debuggear | Mejor uso de recursos, aguanta muchas más llamadas de I/O en simultáneo (ej: red, disco, base de datos) |
| En contra | Desperdicia recursos mientras espera, no escala bien con muchas llamadas simultáneas | Más difícil de razonar y debuggear si algo sale mal |

Cuidado con un mito común: async **no siempre es más rápido**. Sirve para operaciones que esperan (red, disco, base de datos). Para cálculos puros de CPU (sin espera de por medio), no aporta ninguna ventaja, y puede incluso agregar overhead.

### 8. Acoplamiento (coupling) y cohesión (cohesion)

**Cohesión** = qué tan relacionadas están las cosas que viven juntas dentro de una misma clase o módulo. **Acoplamiento** = qué tan dependiente es una clase o módulo de los detalles internos de otro.

**Ejemplo:** una clase `PedidoService` que calcula el precio, envía el email de confirmación, actualiza el stock Y genera la factura, todo junto, tiene **baja cohesión** — son responsabilidades distintas mezcladas en un solo lugar. Si además esa clase necesita conocer los detalles internos de `EmailService`, `StockService` y `FacturaService` para funcionar, tiene **alto acoplamiento** con todas ellas — cualquier cambio en una de esas clases puede romper `PedidoService`.

El objetivo siempre es el mismo, en cualquier nivel (una clase, un servicio, un microservicio): **alta cohesión, bajo acoplamiento**. Cosas relacionadas, juntas. Cosas independientes, sueltas entre sí.

## Otros trade-offs que vas a escuchar

- **Consistencia vs. disponibilidad** (el famoso "teorema CAP"): en un sistema distribuido, si parte de la red falla, tienes que elegir entre seguir respondiendo aunque algunos datos estén desactualizados (disponibilidad) o dejar de responder hasta garantizar que el dato es 100% correcto (consistencia). No puedes tener ambas garantizadas al 100% al mismo tiempo durante una falla de red.
- **Microservicios vs. monolito**
- **Usar una herramienta de terceros vs. construirla tú mismo (in-house)**
- **Cloud vs. servidores propios (on-premise)**
- **Seguridad vs. usabilidad** (ej: pedir 2FA en cada login es más seguro, pero más incómodo para el usuario)
- **Optimizar para lectura vs. para escritura** en una base de datos

## Framework práctico para navegar trade-offs

Cuando tengas que decidir entre dos opciones, esta es una guía simple:

1. **Piensa en el corto y el largo plazo.** Prioriza el largo plazo — lo que parece la solución rápida hoy, muchas veces sale más caro dentro de 6 meses.
2. **Identifica quién depende de esta decisión** (tu equipo, otro equipo, el cliente) y qué necesita cada uno. No vas a poder conformar a todos — prioriza a los que más importan para que la decisión sea aprobada.
3. **Investiga antes de decidir.** Los requerimientos casi nunca están 100% claros al principio — pregunta, arma un boceto de diseño, muéstralo, ajusta.
4. **Documenta la decisión y por qué la tomaste.** Esto se llama **ADR** (Architecture Decision Record) — un documento corto con: qué elegiste, por qué, qué otras opciones consideraste, y por qué las descartaste. Le ahorra dolores de cabeza a quien continúe el proyecto después (incluido tú mismo, dentro de 6 meses).
5. **Si tienes dudas entre dos opciones extremas, prueba un prototipo pequeño de cada una** antes de comprometerte. Es mucho más barato descartar una idea en una prueba de 2 días que después de construir el sistema completo.
6. **Empieza simple, optimiza después.** Construye la solución más simple que cumpla lo que se necesita HOY. Recién ahí, si hace falta, hazla más flexible o más rápida.

## Nota práctica

No existe el sistema perfecto — existe el sistema bien equilibrado para el contexto en el que vive. Ser Semi-Senior en esto no es memorizar los 8 trade-offs de esta lista: es agarrar el hábito de preguntarte, antes de decidir, **"¿qué estoy sacrificando con esta elección, y tiene sentido ese sacrificio para este proyecto puntual?"**
