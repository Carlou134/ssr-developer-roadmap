# ¿Qué son los trade-offs?

> Basado en el artículo "Software Engineering Trade-Offs".

## En una frase

Un trade-off es **lo que sacrificas para ganar otra cosa**: en software no existe la opción que sea a la vez la más simple, la más rápida, la más flexible y la más barata, así que decidir bien es elegir qué priorizar según el contexto y dejar claro qué se pierde.

-----

## Antes de empezar

Conviene que ya sepas:

* Por qué se evalúan 2-3 alternativas antes de escalar: [Investigación y criterio](01-investigacion-y-criterio.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Trade-off (compromiso):** decisión en la que mejorar un aspecto implica empeorar otro.
* **Deuda técnica:** lo que se deja "para después" (tests, diseño, seguridad, observabilidad) para avanzar más rápido ahora, y que después cuesta más resolver.
* **Time-to-market:** cuánto se tarda en poner algo en manos de los usuarios.
* **Escalado vertical / horizontal:** hacer más potente una máquina frente a agregar más máquinas que se reparten el trabajo.
* **Latencia:** cuánto tarda en responder una petición puntual.
* **Throughput:** cuántas peticiones se procesan en total por unidad de tiempo.
* **Stateful / stateless:** el servidor recuerda información entre peticiones frente a que cada petición trae todo lo que necesita.
* **Acoplamiento:** cuánto depende un módulo de los detalles internos de otro.
* **Cohesión:** cuán relacionadas están las responsabilidades que viven juntas en un mismo módulo.
* **SLO (*Service Level Objective*):** objetivo medible de calidad de servicio, como "el 99 % de las peticiones responde en menos de 200 ms".
* **ADR (*Architecture Decision Record*):** documento corto que registra una decisión técnica, sus alternativas y por qué se eligió.

-----

## El problema

Un equipo arma un sistema de notificaciones. Hoy solo se envían emails. Un desarrollador propone:

```csharp
public interface INotificationSender { Task EnviarAsync(Notificacion n); }
public class EmailSender : INotificationSender { /* ... */ }
public class SmsSender : INotificationSender { /* sin usar */ }
public class PushSender : INotificationSender { /* sin usar */ }
public class SlackSender : INotificationSender { /* sin usar */ }
public class NotificationSenderFactory { /* elige el sender según configuración */ }
```

"Por si algún día se necesita". La discusión en el PR dura dos días: unos dicen "esto es sobreingeniería", otros "esto es buen diseño". Nadie dice **qué gana y qué pierde** cada opción, así que gana quien insiste más.

Seis meses después, nunca se usó SMS ni Slack, y el único canal nuevo que pidieron fue WhatsApp, que no encaja en la interfaz porque necesita plantillas aprobadas.

Las consecuencias:

* se discute "bueno o malo" en lugar de "qué sacrificamos y si nos conviene";
* se escribe y se mantiene código para escenarios que nunca llegan;
* la decisión no queda registrada, y el próximo que llegue no sabrá por qué el código está así.

-----

## Cómo funciona

### 1. No existe el sistema perfecto

Es como comprar un auto: no existe el más barato, el más rápido y el que menos combustible gasta, todo junto. Eliges según lo que más te importa.

Un sistema ideal cumpliría cada requerimiento, manejaría todos los casos raros y además sería simple de mantener. En la vida real no pasa, porque:

* cada caso nuevo que el sistema soporta le agrega complejidad;
* cada librería o herramienta nueva, también;
* si priorizas que algo sea simple de mantener, probablemente sacrificas flexibilidad futura, y viceversa.

La buena noticia: en software casi nada es 100 % irreversible. En el peor caso se puede reescribir una parte del sistema. Cuesta tiempo y dinero, pero es una salida real.

Cuando evalúas 2-3 alternativas antes de escalar un problema ([Investigación y criterio](01-investigacion-y-criterio.md)), lo que estás haciendo es exactamente esto: comparar qué gana y qué pierde cada opción.

### 2. Los 8 trade-offs principales

| # | Trade-off | La pregunta que te haces |
| --- | --- | --- |
| 1 | Complejidad frente a todo lo demás | ¿Esto justifica la complejidad que agrega? |
| 2 | Simplicidad frente a flexibilidad | ¿Lo necesito hoy o "por si acaso"? |
| 3 | Time-to-market frente a deuda técnica | ¿Cuánto me cuesta después lo que me ahorro hoy? |
| 4 | Escalado vertical frente a horizontal | ¿Una máquina más grande alcanza? |
| 5 | Latencia frente a throughput | ¿Importa más la respuesta individual o el volumen total? |
| 6 | Stateful frente a stateless | ¿Dónde vive el estado si hay varias instancias? |
| 7 | Síncrono frente a asíncrono | ¿La operación espera I/O? |
| 8 | Acoplamiento y cohesión | ¿Lo que está junto cambia junto? |

#### 1. Complejidad frente a todo lo demás

Casi cualquier decisión agrega complejidad. El objetivo no es "cero complejidad" (imposible), sino mantenerla lo más baja posible para lo que el proyecto realmente necesita.

**Ejemplo:** tu API exporta datos en CSV. Te piden Excel. Después PDF. Después XML. Cada formato nuevo es más código, más pruebas y más lugares donde algo se puede romper. En algún punto hay que preguntarse: ¿de verdad necesitamos los 4 formatos o alcanza con 2?

**Costo de la complejidad alta:** cuesta más mantener el sistema, cuesta más que alguien nuevo lo entienda (*onboarding*), cuesta más cada cambio y, en el peor caso, el sistema se vuelve tan complicado que da miedo tocarlo.

#### 2. Simplicidad frente a flexibilidad

* **Simplicidad:** código fácil de entender y mantener hoy.
* **Flexibilidad:** código preparado para adaptarse a cambios futuros.

**Ejemplo:** el sistema de notificaciones del problema. Puedes escribir una clase simple que envía el email (si mañana piden SMS, hay que modificarla) o armar una interfaz genérica con implementaciones intercambiables "por si algún día se necesita" (flexible, pero hoy es código que nadie usa y que cuesta entender).

| | Simplicidad | Flexibilidad |
| --- | --- | --- |
| A favor | Se entiende rápido, hay menos cosas que puedan fallar | Es más fácil de extender cuando el cambio realmente llega |
| En contra | Puede requerir reescribir si el requerimiento cambia antes de lo esperado | El código "por las dudas" infla el proyecto y es más difícil de probar |

Regla de oro: **No Big Design Up Front**. No diseñes hoy para todos los escenarios que imaginas: la mitad nunca va a pasar y la otra mitad va a ser distinta de como la imaginaste.

#### 3. Time-to-market frente a deuda técnica

* **Time-to-market:** qué tan rápido lanzas algo.
* **Deuda técnica:** todo lo que dejas "para después" para lanzar más rápido ahora.

**Ejemplo:** tu equipo tiene 2 semanas para el MVP. Puedes escribir pruebas completas y una arquitectura prolija (tarda más, pero es sólido) o saltarte las pruebas y dejar valores fijos en el código para llegar a tiempo (rápido ahora, pero cada bug futuro cuesta más).

| | Lanzar rápido | Invertir en calidad ahora |
| --- | --- | --- |
| A favor | Llegas antes a los usuarios, obtienes feedback real y generas ingresos antes | Menos bugs y caídas, las siguientes funcionalidades se construyen más rápido |
| En contra | La deuda se acumula y cuesta más pagarla después; más riesgo de bugs y de problemas de seguridad | Lanzas más tarde; un competidor puede ganarte el mercado mientras pules |

> **Ojo:** deuda técnica no es solo "código feo". No tener pruebas, no tener logs o no tener la seguridad revisada también es deuda técnica, aunque el código se vea prolijo.

#### 4. Escalado vertical frente a horizontal

* **Vertical:** hacer más potente la misma máquina (más RAM, más CPU).
* **Horizontal:** agregar más máquinas que se reparten el trabajo.

**Ejemplo:** tu API recibe más tráfico del que soporta. Puedes subir de plan el servidor (vertical: simple, pero tiene un techo) o poner 3 instancias detrás de un balanceador de carga (horizontal: escala casi sin límite, pero ahora tienes que pensar qué pasa si un usuario llega a la instancia 1 y después a la 2: ¿comparten sesión?).

| | Vertical | Horizontal |
| --- | --- | --- |
| A favor | Más simple de operar, no hay que coordinar varias máquinas | Escala casi sin límite; si una máquina se cae, las otras siguen |
| En contra | Tiene un techo y es un único punto de falla | Más complejo de operar; el estado tiene que poder compartirse entre instancias |

Cambiar de un modelo al otro más adelante no es gratis: normalmente implica repensar buena parte de la arquitectura.

#### 5. Latencia frente a throughput

* **Latencia:** cuánto tarda en responder una petición puntual.
* **Throughput:** cuántas peticiones en total procesas por segundo.

Parece que optimizar uno mejora el otro automáticamente, pero no siempre es así. Optimizar latencia es dedicar recursos extra a cada petición para que responda rápido. Optimizar throughput es repartir esos recursos entre muchas peticiones, priorizando el volumen total sobre la velocidad de una sola.

**Ejemplo:** un buscador que responde en 50 ms está optimizado en latencia. Un proceso por lotes que procesa 10 000 pagos por minuto, aunque cada pago tarde 2 segundos, está optimizado en throughput.

Términos que vas a escuchar seguido:

* **SLO:** un objetivo concreto, por ejemplo "el 99 % de las peticiones responde en menos de 200 ms".
* **p99 (percentil 99):** el 99 % de las peticiones responde en ese tiempo o menos; el 1 % restante puede tardar más.
* **Back-pressure (contrapresión):** cuando llega más trabajo del que el sistema puede procesar y empieza a acumularse, como una cola que no para de crecer.

Recomendación práctica: si no tienes claro cuál priorizar, define primero un SLO (por ejemplo, "p99 menor a 300 ms") y después optimiza throughput sin romper ese límite.

#### 6. Stateful frente a stateless

* **Stateful:** el servidor guarda información sobre lo que el usuario venía haciendo.
* **Stateless:** cada petición trae toda la información que necesita; el servidor no recuerda nada entre una y otra.

**Ejemplo:** un carrito guardado en la memoria del servidor (stateful) es rápido y simple mientras el usuario llegue siempre a la misma instancia. Pero si escalas horizontalmente y el usuario cae en otra instancia, el carrito "desaparece". Guardarlo en una base de datos o en Redis, accesible desde cualquier instancia, lo resuelve a cambio de una llamada de red extra por petición.

| | Stateful | Stateless |
| --- | --- | --- |
| A favor | Más simple de programar, menos llamadas de red | Escala mejor; si una instancia se cae, no se pierde nada |
| En contra | Difícil de escalar horizontalmente; si el servidor se cae, se pierde el estado | El estado viaja por la red o vive en otro sistema; más piezas que coordinar |

#### 7. Síncrono (bloqueante) frente a asíncrono (no bloqueante)

* **Síncrono:** tu código espera a que termine una operación (por ejemplo, una llamada a una API externa) antes de seguir.
* **Asíncrono:** el hilo queda libre para otras tareas mientras la operación termina.

```csharp
// Bloqueante: el hilo queda esperando
var response = httpClient.GetAsync(url).Result;

// No bloqueante: el hilo queda libre mientras espera
var response = await httpClient.GetAsync(url);
```

| | Síncrono | Asíncrono |
| --- | --- | --- |
| A favor | Más fácil de leer y depurar | Mejor uso de recursos; soporta muchas más operaciones de I/O en simultáneo (red, disco, base de datos) |
| En contra | Desperdicia hilos mientras espera; no escala bien con muchas llamadas simultáneas | Más difícil de razonar y depurar cuando algo sale mal |

> **Ojo:** el `.Result` del ejemplo no es solo "síncrono": es bloquear código asíncrono (*sync over async*). En ASP.NET Core puede agotar el pool de hilos bajo carga, y en aplicaciones con contexto de sincronización (WinForms, WPF, ASP.NET clásico) puede causar un *deadlock*. Si la API es asíncrona, usa `await` de punta a punta.

Mito común: async **no siempre es más rápido**. Sirve para operaciones que esperan (red, disco, base de datos). Para cálculos puros de CPU no aporta ventaja y puede agregar overhead.

#### 8. Acoplamiento y cohesión

* **Cohesión:** cuán relacionadas están las cosas que viven juntas en una misma clase o módulo.
* **Acoplamiento:** cuánto depende una clase o módulo de los detalles internos de otro.

**Ejemplo:** un `PedidoService` que calcula el precio, envía el email de confirmación, actualiza el stock y genera la factura tiene **baja cohesión**: mezcla responsabilidades distintas. Si además necesita conocer los detalles internos de `EmailService`, `StockService` y `FacturaService`, tiene **alto acoplamiento**: cualquier cambio en una de ellas puede romperlo.

El objetivo es el mismo en cualquier nivel (clase, servicio, microservicio): **alta cohesión, bajo acoplamiento**. Lo que está relacionado, junto. Lo que es independiente, separado.

### 3. Otros trade-offs que vas a escuchar

* **Consistencia frente a disponibilidad (teorema CAP):** en un sistema distribuido, si parte de la red falla, tienes que elegir entre seguir respondiendo aunque algunos datos estén desactualizados (disponibilidad) o dejar de responder hasta garantizar que el dato es correcto (consistencia). Durante una partición de red no puedes garantizar ambas.
* **Microservicios frente a monolito.**
* **Usar una herramienta de terceros frente a construirla en casa (*build vs. buy*).**
* **Cloud frente a servidores propios (*on-premise*).**
* **Seguridad frente a usabilidad:** pedir 2FA en cada login es más seguro, pero más incómodo.
* **Optimizar para lectura frente a optimizar para escritura** en una base de datos.

### 4. Framework práctico para decidir

1. **Piensa en el corto y en el largo plazo.** Lo que parece la solución rápida hoy muchas veces sale más caro dentro de 6 meses.
2. **Identifica quién depende de la decisión** (tu equipo, otro equipo, el cliente) y qué necesita cada uno. No vas a poder conformar a todos: prioriza a quienes más importan para el proyecto.
3. **Investiga antes de decidir.** Los requerimientos casi nunca están 100 % claros al principio: pregunta, arma un boceto, muéstralo y ajusta.
4. **Documenta la decisión y el porqué en un ADR:** qué elegiste, por qué, qué otras opciones consideraste y por qué las descartaste. Le ahorra dolores de cabeza a quien continúe el proyecto, incluido tú dentro de 6 meses.
5. **Si dudas entre dos opciones extremas, prueba un prototipo pequeño de cada una.** Descartar una idea en una prueba de 2 días es mucho más barato que hacerlo después de construir el sistema completo.
6. **Empieza simple y optimiza después.** Construye la solución más simple que cumple lo que se necesita hoy. Recién después, si hace falta, hazla más flexible o más rápida.

-----

## Ejemplo completo

La decisión del sistema de notificaciones, tomada con el framework.

**1. Corto y largo plazo.** Hoy: solo email. A futuro: producto menciona "quizás push en algún momento", sin fecha ni requerimiento concreto.

**2. Quién depende.** El equipo que lo mantiene (quiere poco código) y producto (quiere poder agregar canales sin una reescritura grande).

**3. Investigar.** Se pregunta a producto: no hay ningún canal nuevo en el roadmap de los próximos dos trimestres.

**4. Alternativas.**

| | A. Clase simple `EmailNotifier` | B. Interfaz + 4 implementaciones + factory |
| --- | --- | --- |
| Código hoy | 1 clase | 6 tipos, 3 sin uso |
| Agregar un canal | Extraer una interfaz y crear la nueva clase (horas) | Crear la nueva clase (minutos), si encaja en la interfaz |
| Riesgo | Refactor pequeño cuando llegue el canal | Mantener código muerto y una abstracción que puede no encajar |

**5. Decisión.** Opción A: hoy no hay requerimiento que justifique la flexibilidad, y extraer la interfaz después es un refactor chico y seguro.

**6. ADR.**

```markdown
# ADR-007: Notificaciones con una clase simple de email

## Estado
Aceptada (2026-10-25)

## Contexto
Hoy solo se envían emails. No hay canales nuevos en el roadmap de los próximos dos trimestres.

## Decisión
Implementar `EmailNotifier` como clase concreta, sin interfaz genérica de canales.

## Alternativas consideradas
- Interfaz `INotificationSender` con email, SMS, push y Slack: descartada porque
  3 de 4 implementaciones no tienen uso y la abstracción podría no encajar con
  canales reales futuros.

## Consecuencias
- Menos código para mantener y entender.
- Cuando llegue un segundo canal, habrá que extraer una interfaz (refactor estimado: medio día).
```

Qué observar:

* La discusión dejó de ser "buen diseño frente a sobreingeniería" y pasó a ser "qué ganamos y qué perdemos con cada opción".
* La decisión **no es para siempre**: el ADR deja escrito cuándo revisarla (cuando llegue un segundo canal).
* Si WhatsApp llega con sus plantillas aprobadas, la interfaz se diseña con **dos** casos reales, no con cuatro imaginados.

-----

## Errores comunes

**1. Buscar la solución perfecta.**
Qué pasa: la decisión se posterga días porque todas las opciones tienen algo malo.
Por qué: se busca una opción sin desventajas, y no existe.
Arreglo: haz explícito qué sacrifica cada opción y elige la que sacrifica lo que menos importa en este contexto.

**2. Diseñar "por si acaso".**
Qué pasa: abstracciones, capas y configuraciones que nadie usa.
Por qué: se optimiza para escenarios imaginados en lugar de para requerimientos reales.
Arreglo: No Big Design Up Front; construye para hoy y deja el código fácil de cambiar.

**3. Copiar la decisión de otra empresa.**
Qué pasa: microservicios, Kubernetes y colas de mensajes para un equipo de 3 personas con 100 usuarios.
Por qué: se copia la solución sin copiar el contexto que la justificó (escala, equipo, presupuesto).
Arreglo: evalúa los trade-offs con **tus** restricciones, no con las de Netflix.

**4. No documentar la decisión.**
Qué pasa: meses después, alguien "arregla" una decisión deliberada porque parecía un error.
Por qué: el porqué quedó en una reunión o en un chat.
Arreglo: un ADR corto por cada decisión que no sea obvia.

**5. Confundir async con rápido.**
Qué pasa: se convierte en `async` un cálculo de CPU y no mejora nada, o empeora.
Por qué: async libera el hilo mientras **espera**; si no hay espera, no hay nada que liberar.
Arreglo: async para I/O; para CPU, optimiza el algoritmo o paraleliza.

-----

## Cuándo sí y cuándo no

**Haz un análisis explícito de trade-offs cuando:** la decisión es difícil de revertir (base de datos, arquitectura, proveedor), afecta a otros equipos o el equipo no se pone de acuerdo.

**Escribe un ADR cuando:** alguien nuevo podría preguntarse "¿por qué está hecho así?" y la respuesta no está en el código.

**No hagas un análisis formal cuando:** la decisión es fácil de revertir y de bajo impacto, como el nombre de una variable privada o qué librería usar en un script interno. Ahí decide rápido y sigue.

-----

## Resumen en 5 líneas

1. Un trade-off es lo que sacrificas para ganar otra cosa; no existe la opción sin desventajas.
2. Los más comunes: complejidad, simplicidad frente a flexibilidad, velocidad frente a deuda técnica, escalado, latencia frente a throughput, estado, sincronía y acoplamiento.
3. Empieza simple: No Big Design Up Front.
4. Decide con el contexto propio, pensando en corto y largo plazo y en quién depende de la decisión.
5. Documenta las decisiones no obvias en un ADR, con las alternativas descartadas.

-----

## Para profundizar

<details>
<summary>Decisiones de una vía y de dos vías</summary>

Una forma rápida de saber cuánto analizar: ¿la decisión se puede revertir fácilmente?

* **Dos vías (reversible):** una librería de UI, la estructura de una clase, un flag de configuración. Decide rápido; si te equivocas, vuelves atrás.
* **Una vía (difícil de revertir):** el motor de base de datos, el proveedor de cloud, un contrato público de API que consumen otros. Aquí sí vale la pena el análisis completo, el prototipo y el ADR.

La mayoría de las decisiones del día a día son de dos vías. Tratarlas como si fueran de una vía es otra forma de perder tiempo.

</details>

<details>
<summary>YAGNI y la regla de tres</summary>

**YAGNI** (*You Aren't Gonna Need It*) es la versión corta de No Big Design Up Front: no implementes algo hasta que realmente lo necesites.

La **regla de tres** ayuda a decidir cuándo abstraer: la primera vez escribes el código; la segunda, duplicas con cierta incomodidad; la tercera, extraes la abstracción. Con tres casos reales, la abstracción se diseña sobre evidencia y no sobre suposiciones.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un trade-off es cuando, para ganar algo, tienes que sacrificar otra cosa. Por ejemplo, lanzar rápido a cambio de acumular deuda técnica, o hacer el código más flexible a cambio de hacerlo más complejo.

### Respuesta ampliada (semi-senior)

Para mí, decidir bien no es encontrar la opción perfecta, sino hacer explícito qué sacrifica cada alternativa y elegir según el contexto: plazos, equipo, escala y cuán reversible es la decisión. Empiezo por la solución más simple que cumple el requerimiento actual y evito diseñar para escenarios imaginados. En las decisiones difíciles de revertir comparo 2-3 alternativas, a veces con un prototipo, y registro la decisión en un ADR con las opciones descartadas. Por ejemplo, prefiero un monolito bien modularizado antes que microservicios si el equipo es chico, sabiendo que pierdo escalado independiente a cambio de operar mucho menos.

### Preguntas frecuentes de seguimiento

**1. Cuéntame una decisión técnica con un trade-off que hayas tomado.**
Estructura: el contexto, las 2-3 alternativas, qué sacrificaba cada una, qué elegiste y por qué, y cómo resultó. Lo importante es mostrar que el sacrificio fue consciente.

**2. ¿Cómo decides entre lanzar rápido o hacerlo bien?**
Depende de cuán reversible sea y del costo de equivocarse. En un MVP para validar una idea, priorizo velocidad, pero dejo registrada la deuda y no omito lo que sería caro agregar después, como la seguridad o un mínimo de observabilidad.

**3. ¿Qué es un ADR?**
Un documento corto que registra una decisión de arquitectura: contexto, decisión, alternativas consideradas y consecuencias. Sirve para que quien llegue después entienda el porqué.

-----

## Práctica

**Ejercicio 1.** Tu API corre en un solo servidor que llega al 90 % de CPU en horas pico. El tráfico crece un 10 % por mes. Guarda la sesión de los usuarios en memoria. ¿Qué trade-offs aparecen si decides escalar horizontalmente?

<details>
<summary>Solución</summary>

* **Vertical frente a horizontal:** el horizontal escala mejor con un crecimiento sostenido del 10 % mensual, pero es más complejo de operar (balanceador, despliegues en varias instancias).
* **Stateful frente a stateless:** con la sesión en memoria, un usuario que cambia de instancia pierde su sesión. Hay que moverla a Redis o a la base de datos (una llamada de red extra por petición) o usar *sticky sessions* (más simple, pero si una instancia cae, sus usuarios pierden la sesión).
* **Time-to-market:** escalar verticalmente da tiempo hoy; puede ser razonable hacerlo primero y planificar la migración a stateless con calma.

</details>

**Ejercicio 2.** ¿Qué tiene de malo este argumento en una revisión de código? ¿Cómo lo reformularías?

```text
"Esto hay que hacerlo con microservicios porque es la mejor práctica.
Así lo hace Netflix."
```

<details>
<summary>Solución</summary>

* "Mejor práctica" sin contexto: no hay opción mejor en abstracto, solo mejor para un contexto.
* Copia la decisión de una empresa con otra escala, otro equipo y otro presupuesto.
* No menciona qué se sacrifica: complejidad operativa, transacciones distribuidas, observabilidad, latencia de red entre servicios.

Reformulado:

```text
"Propongo separar Pagos en un servicio aparte porque su carga es 10 veces la del
resto y hoy no podemos escalarlo de forma independiente. A cambio sumamos una
llamada de red y un despliegue más que mantener. La alternativa es mantener el
monolito y escalarlo completo, que hoy cuesta X por mes. ¿Lo vemos con números?"
```

</details>

-----

## Siguiente lección

Terminaste las lecciones de esta fase. Para pedir ayuda estructurada con lo que aprendiste, usa los [Prompts útiles](06-prompts-utiles.md). Para continuar: [Calidad de código y arquitectura](../02-calidad-de-codigo-y-arquitectura/).
