# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Acoplamiento (*coupling*).** Cuánto depende un módulo de los detalles internos de otro. Se busca que sea bajo.

**ADR (*Architecture Decision Record*).** Documento corto que registra una decisión técnica: contexto, decisión, alternativas consideradas y consecuencias.

**Breakpoint.** Marca en una línea donde el depurador pausa la ejecución.

**Breakpoint condicional.** Breakpoint que solo pausa cuando se cumple una expresión o al llegar a cierta cantidad de pasadas.

**Búsqueda binaria.** Estrategia que descarta la mitad de los candidatos en cada paso. Es la base de `git bisect`.

**Call stack (pila de llamadas).** Lista de métodos que se fueron llamando hasta llegar al punto actual de la ejecución. Funciona como LIFO.

**Categoría (de log).** Nombre que identifica el origen de un log. Con `ILogger<T>` es el nombre completo del tipo `T`.

**CLR (*Common Language Runtime*).** El motor de .NET que ejecuta el código y maneja las excepciones.

**Cohesión.** Cuán relacionadas están las responsabilidades que viven juntas en un mismo módulo. Se busca que sea alta.

**Correlation ID.** Identificador único que acompaña a una transacción a través de todos los servicios que toca, para poder reunir sus logs.

**Desenrollado de la pila (*stack unwinding*).** Proceso de sacar frames de la pila hasta llegar al `catch` que va a manejar la excepción, ejecutando los `finally` del camino.

**Deuda técnica.** Lo que se deja "para después" (pruebas, diseño, seguridad, observabilidad) para avanzar más rápido ahora, y que después cuesta más resolver.

**Downstream.** Servicios a los que llama el sistema actual, más abajo en la cadena de una petición.

**Escalado horizontal.** Agregar más máquinas que se reparten el trabajo.

**Escalado vertical.** Hacer más potente la misma máquina: más CPU, más memoria.

**Escalar (un problema).** Pedir ayuda a alguien con más experiencia o autoridad cuando no puedes resolver un bloqueo solo.

**Excepción interna (*inner exception*).** Excepción original que otra envuelve para agregar contexto. Se consulta con `InnerException` y aparece después de `--->` en el stack trace.

**Excepción no controlada.** Excepción que ningún `catch` atrapa y llega al punto de entrada del programa.

**Filtro de excepción (`when`).** Condición que decide si un `catch` atrapa o no una excepción. Se evalúa antes de desenrollar la pila.

**Frame.** Cada entrada del call stack: un método en ejecución con sus variables.

**`git bisect`.** Comando que aplica búsqueda binaria sobre el historial para encontrar el commit que introdujo un bug.

**`git blame`.** Comando que muestra quién modificó cada línea de un archivo por última vez y en qué commit.

**Hit count.** Cantidad de veces que la ejecución pasó por un breakpoint.

**Latencia.** Cuánto tarda en responder una petición puntual.

**LIFO (*Last In, First Out*).** El último elemento que entra es el primero que sale. Así funciona la pila de llamadas.

**Log estructurado.** Registro con campos con nombre (`UserId`, `PedidoId`) en lugar de solo texto libre, para poder filtrarlo y consultarlo.

**Memory leak (fuga de memoria).** Memoria que la aplicación ya no necesita pero que no se libera.

**MRE (*Minimal Reproducible Example*).** El caso más chico posible que sigue reproduciendo el bug.

**Nivel de log (*log level*).** Severidad de un registro, de `Trace` (0) a `Critical` (5). El nivel configurado actúa como piso.

**Plantilla de mensaje (*message template*).** Texto de log con placeholders con nombre, como `"Login fallido para {UserId}"`.

**Profiler.** Herramienta que mide en qué se gasta el tiempo de CPU o la memoria.

**Propagación.** La excepción sube de un método a quien lo llamó hasta encontrar un `catch` compatible.

**Rabbit hole.** Investigación que te va llevando cada vez más lejos sin acercarte a la solución.

**Regresión.** Algo que funcionaba y dejó de funcionar por un cambio.

**Relanzar (*rethrow*).** Volver a lanzar desde un `catch` la excepción atrapada. En .NET se hace con `throw;` para conservar el stack trace.

**Sink.** Destino al que Serilog envía los logs: consola, archivo, base de datos, Seq.

**SLO (*Service Level Objective*).** Objetivo medible de calidad de servicio, como "el 99 % de las peticiones responde en menos de 200 ms".

**Stack trace (traza de la pila).** Lista de los métodos que estaban en la pila de llamadas cuando se lanzó una excepción. En .NET se lee de arriba hacia abajo.

**Stateful.** El servidor guarda información sobre lo que el usuario venía haciendo entre una petición y otra.

**Stateless.** Cada petición trae toda la información que necesita; el servidor no recuerda nada entre una y otra.

**Step over / into / out.** Comandos del depurador para avanzar a la siguiente línea, entrar al método que se llama o salir del método actual.

**Throughput.** Cuántas peticiones se procesan en total por unidad de tiempo.

**Time-to-market.** Cuánto se tarda en poner algo en manos de los usuarios.

**Timeboxing.** Fijar de antemano cuánto tiempo vas a dedicar a algo antes de cambiar de estrategia o escalar.

**Tracepoint (*logpoint*).** Breakpoint que escribe un mensaje en la consola de depuración sin pausar la ejecución.

**Trade-off (compromiso).** Decisión en la que mejorar un aspecto implica empeorar otro.
