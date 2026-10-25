# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Abstracción.** Tipo (interfaz o clase abstracta) que define *qué* se hace sin decir *cómo*.

**Abstracción incorrecta.** Código compartido que une casos que en realidad cambian por razones distintas. Suele llenarse de flags booleanos.

**Adaptador (*adapter*).** En arquitectura hexagonal, la pieza concreta que conecta una tecnología con un puerto.

**Adaptador primario (*driving adapter*).** Adaptador que llama hacia adentro del núcleo, como un controller REST, un consumidor de colas o un job.

**Adaptador secundario (*driven adapter*).** Adaptador que implementa un puerto secundario y sale hacia afuera, como un repositorio JPA o un cliente HTTP.

**Análisis estático.** Revisión automática del código sin ejecutarlo, como la que hacen SonarQube o los analizadores de Roslyn.

**BDD (*Behavior-Driven Development*).** Describir el comportamiento esperado en lenguaje de negocio, a menudo con Given/When/Then.

**Big Ball of Mud (gran bola de barro).** Sistema sin estructura reconocible, donde todo depende de todo. Es un antipatrón de arquitectura.

**Binding.** Asignar automáticamente los valores de la configuración a las propiedades de una clase.

**BLL (*Business Logic Layer*).** La capa de lógica de negocio en una arquitectura en capas.

**Bug.** Error o defecto que hace que el sistema se comporte de forma distinta a la esperada.

**Cableado (*wiring*).** Construir los objetos concretos y pasarlos donde se piden sus interfaces.

**Capa.** Grupo de código con una responsabilidad común que solo se comunica con otras capas por puntos definidos.

**CI (*Continuous Integration*).** Ejecutar automáticamente la compilación y las pruebas en cada cambio que se sube al repositorio.

**Clase abstracta.** Clase que no se puede instanciar y sirve de base para otras; puede tener estado, constructores y métodos implementados.

**Clase Dios (*God Class*).** Clase que hace de todo y por la que termina pasando cualquier cambio del sistema.

**Cobertura de código.** Porcentaje del código que se ejecutó durante las pruebas. No indica si se verificó algo.

**Code smell (olor de código).** Señal superficial que sugiere un problema de diseño más profundo, aunque el código funcione.

**Command.** Petición que cambia el estado del sistema. También es el patrón que encapsula una petición en un objeto.

**Complejidad ciclomática.** Cantidad de caminos independientes que tiene un método; cuantos más, más difícil de entender y de probar.

**Composition root.** El único lugar donde se conectan las interfaces con sus implementaciones; en ASP.NET Core, `Program.cs`.

**Consistencia eventual.** Los datos de lectura se actualizan un poco después que los de escritura, no en el mismo instante.

**Contenedor de DI.** Componente que registra qué implementación corresponde a cada abstracción y crea las instancias cuando alguien las pide.

**CQRS (*Command Query Responsibility Segregation*).** Modelos separados para escribir y para leer. No exige dos bases de datos.

**CQS (*Command-Query Separation*).** Un método cambia estado o devuelve datos, nunca las dos cosas.

**DAL (*Data Access Layer*).** La capa de acceso a datos en una arquitectura en capas.

**Default interface method.** Método de una interfaz con una implementación por defecto. Existe desde C# 8.

**Dependencia cautiva (*captive dependency*).** Dependencia de vida corta (por ejemplo, Scoped) atrapada dentro de un servicio de vida más larga (Singleton).

**Despacho dinámico.** Decidir en tiempo de ejecución qué implementación de un método se llama, según el tipo real del objeto.

**Deuda técnica.** El costo extra que se paga en cada cambio futuro por un atajo tomado hoy.

**Domain Event (evento de dominio).** Aviso de que algo importante pasó en el negocio, como `PedidoCreado`.

**Dominio.** Las reglas y conceptos del negocio, independientes de la tecnología.

**DRY (*Don't Repeat Yourself*).** Cada pieza de conocimiento debe tener una única representación en el sistema.

**DTO (*Data Transfer Object*).** Objeto pensado solo para transportar datos entre capas o hacia afuera, sin lógica.

**E2E (*end-to-end*), prueba.** Prueba del flujo completo, tal como lo usaría un usuario.

**Entities (entidades).** En Clean Architecture, los objetos con las reglas de negocio más generales.

**Escuela clásica / londinense (de pruebas unitarias).** En la clásica (Detroit), la unidad es un comportamiento y solo se reemplaza lo externo. En la londinense (*mockist*), cada clase se prueba sola y todas sus colaboradoras se reemplazan.

**Event Sourcing.** Guardar la secuencia de eventos que ocurrieron en lugar del estado actual.

**Excepción de negocio.** Excepción propia que representa la violación de una regla de negocio.

**Fake.** Doble de prueba con una implementación simple que funciona de verdad, como un repositorio en memoria.

**Firma.** Nombre de un método más el tipo, la cantidad y el orden de sus parámetros. El tipo de retorno no forma parte.

**Flag booleano (parámetro bandera).** Parámetro `bool` que cambia el comportamiento de un método, como `esContratista`.

**Frameworks & Drivers.** En Clean Architecture, la capa más externa: framework web, base de datos, UI.

**GoF (*Gang of Four*).** Los cuatro autores del libro *Design Patterns* (1994), que catalogó 23 patrones clásicos.

**Guard clause (cláusula de guarda).** Validación al inicio de un método que corta la ejecución si los datos no sirven.

**Handler.** La clase que procesa un command o una query concreta.

**Indirección.** Salto que hay que hacer para leer el comportamiento real, por ejemplo de una interfaz a su implementación.

**Infraestructura.** El código que habla con el mundo exterior: base de datos, archivos, APIs, email.

**Inyección de dependencias.** Pasarle a una clase lo que necesita desde afuera, normalmente por el constructor, en lugar de que lo cree ella.

**Interface Adapters (adaptadores de interfaz).** En Clean Architecture, el código que traduce entre el núcleo y el exterior: controllers, presenters, gateways.

**Interfaz.** Contrato que define qué miembros debe tener un tipo, sin estado de instancia.

**Interfaz gorda (*Fat Interface*).** Interfaz que obliga a implementar métodos que el implementador no necesita.

**Intereses (de la deuda técnica).** El tiempo extra que cuesta cada cambio en el código afectado.

**Invariante.** Condición que siempre debe cumplirse en un objeto, como "el total nunca es negativo".

**KISS (*Keep It Simple, Stupid*).** La solución más simple que funcione suele ser la mejor.

**Lifetime.** Cuánto vive una instancia creada por el contenedor de DI.

**Linter.** Herramienta que analiza el código y marca lo que no sigue las convenciones o reglas configuradas.

**Mediator (mediador).** Objeto que recibe una petición y la entrega al handler que corresponde, sin que quien la envía lo conozca.

**Mock / doble de prueba.** Versión falsa de una dependencia que se usa en una prueba en lugar de la real.

**Modelo de dominio anémico.** Entidades que solo tienen propiedades públicas, con toda la lógica afuera. Es un antipatrón.

**Modelo de dominio rico.** Entidades que protegen sus propias reglas y solo cambian a través de métodos con significado.

**N-Tier (arquitectura en capas).** Organización del código en capas horizontales donde cada una solo usa a la de abajo.

**Número mágico.** Valor fijo en el código sin un nombre que explique qué representa.

**PascalCase / camelCase / snake_case.** Formas de escribir nombres compuestos: `TotalPedido`, `totalPedido`, `total_pedido`.

**Patrón de diseño.** Solución reutilizable a un problema de diseño que aparece una y otra vez.

**Patrón Options.** Leer una sección de configuración como una clase tipada, inyectada con `IOptions<T>`.

**Patrón Result.** Devolver un objeto con el éxito o los errores de una operación, en lugar de lanzar una excepción.

**Pirámide de testing.** Muchas pruebas unitarias en la base, menos de integración y pocas E2E en la punta.

**Proyección.** Consulta que devuelve solo los campos necesarios, directamente como DTO.

**Prueba de integración.** Prueba que ejercita varias piezas reales juntas, como un servicio con su base de datos.

**Prueba unitaria.** Prueba rápida y aislada de una unidad de comportamiento, sin base de datos ni red.

**Puerto (*port*).** En arquitectura hexagonal, interfaz que define un punto de entrada o de salida del núcleo.

**Puerto primario (*driving port*).** Cómo el exterior usa la aplicación, por ejemplo un caso de uso.

**Puerto secundario (*driven port*).** Lo que la aplicación necesita del exterior, por ejemplo un repositorio.

**Quality Gate.** Condiciones de calidad que el código debe cumplir para poder fusionarse.

**Query.** Petición que solo lee datos y no modifica nada.

**Regla de dependencia.** Las dependencias entre capas siguen una sola dirección, hacia el código más estable (el dominio).

**Regla de tres.** Extraer una abstracción recién cuando el mismo código aparece en tres lugares.

**Regla del Boy Scout.** Dejar el código un poco mejor de lo que lo encontraste.

**Regresión.** Algo que funcionaba y dejó de funcionar por un cambio.

**Repositorio genérico.** Un único repositorio `IRepository<T>` reutilizado para todas las entidades.

**Responsabilidad.** Una razón para cambiar. Una clase con tres responsabilidades tiene tres motivos distintos para ser modificada.

**`sealed`.** Modificador que impide heredar de una clase o volver a sobrescribir un método.

**Scope.** Período de vida acotado dentro del cual se comparten las instancias Scoped.

**Scoped.** Lifetime con una instancia por scope; en ASP.NET Core, una por petición HTTP.

**Singleton (lifetime).** Una sola instancia para toda la vida de la aplicación, gestionada por el contenedor de DI.

**Sobrearquitectura.** Una solución más grande o compleja de lo que el problema necesita.

**Sobrecarga (*overloading*).** Varios métodos con el mismo nombre y distintos parámetros. La resuelve el compilador.

**Sobrescritura (*override*).** Una clase hija redefine un método `virtual` o `abstract` de la base. La resuelve el runtime.

**SOLID.** Acrónimo de cinco principios de diseño orientado a objetos: SRP, OCP, LSP, ISP y DIP.

**TDD (*Test-Driven Development*).** Escribir la prueba que falla antes del código, luego el código mínimo y después refactorizar.

**Testcontainers.** Librería que levanta servicios reales (bases de datos, colas) en contenedores Docker durante las pruebas.

**Thread pool.** Conjunto de hilos que .NET reutiliza para atender trabajo, como las peticiones HTTP.

**Transient.** Lifetime con una instancia nueva cada vez que se pide.

**Unidad de trabajo (*Unit of Work*).** Agrupar varios cambios para confirmarlos juntos, todos o ninguno.

**Use Case (caso de uso).** Una operación concreta de la aplicación, como "crear un pedido", que orquesta entidades.

**YAGNI (*You Aren't Gonna Need It*).** No implementes algo hasta que realmente lo necesites.
