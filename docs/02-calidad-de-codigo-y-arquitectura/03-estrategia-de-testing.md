# Estrategia de Testing

## Pruebas manuales vs. automatizadas

Probar software es comparar el comportamiento real contra el esperado. Hay dos formas de hacerlo:

- **Pruebas manuales:** una persona interactúa con la aplicación a mano (clickear, escribir, navegar) y compara lo que ve contra lo que debería pasar. Cuando no coincide, hay un **bug** — un error o defecto que hace que el sistema se comporte de forma inesperada.
- **Pruebas automatizadas:** un programa ejecuta esa comparación en lugar de una persona. Son más rápidas, más confiables (no se cansan ni se distraen) y mantenibles (se revisan y extienden como cualquier otro código).

Un estudio del NIST (Instituto Nacional de Estándares y Tecnología de EE.UU., 2002) estimó el costo de los bugs de software en la economía estadounidense en unos **59.500 millones de dólares anuales** — es una cifra vieja y hoy probablemente distinta, pero ilustra bien la escala real del problema que las pruebas automatizadas buscan reducir.

**Código de prueba vs. código de implementación:** el código que define la aplicación es "código de implementación"; el código que la prueba es "código de prueba". En .NET, conviene organizarlo en un **proyecto de test separado** (ej: `OrderManagementBackend.Tests`), con archivos que reflejan qué testean (`OrderServiceTests.cs` prueba `OrderService.cs`) — no mezclado dentro del proyecto de producción.

## Las pruebas como documentación, y la regresión

Un test bien escrito cumple una función doble: verifica que algo funciona, y describe en lenguaje legible qué se espera que haga esa pieza de código, sin tener que leer la implementación entera para entenderlo. Un nombre de test como `DeberiaRechazarPedidoCompletado_CuandoSeIntentaModificar()` le dice a cualquiera qué regla de negocio existe, sin abrir el código de producción.

**Regresión** es cuando algo que ya funcionaba y ya estaba probado deja de funcionar, generalmente como efecto secundario de un cambio en otra parte del sistema. Correr la suite de tests después de cada cambio es lo que detecta esto — si una funcionalidad "regresó", al menos un test debería fallar y avisar antes de que llegue a producción. Es la razón de fondo por la que la Regla del Boy Scout (ver [04-deuda-tecnica.md](./04-deuda-tecnica.md)) es segura de aplicar: refactorizar sin miedo solo funciona si hay tests que avisan cuando algo se rompe.

## TDD y BDD, en corto

- **TDD (Test-Driven Development):** se escribe el test ANTES que el código de producción. El ciclo es: escribir un test que falla (porque el código todavía no existe) → escribir el código mínimo para que pase → refactorizar con la seguridad de que el test avisa si algo se rompe.
- **BDD (Behavior-Driven Development):** se enfoca en describir el comportamiento esperado en lenguaje cercano al negocio (a menudo con sintaxis Given/When/Then), para que las pruebas también sirvan como documentación legible por alguien no técnico.

No son excluyentes — BDD define el "qué debería pasar" en lenguaje de negocio, TDD es la disciplina de cómo se construye el código para cumplirlo.

**La diferencia concreta, más allá del lenguaje:** en TDD, la unidad mínima que se prueba es una función o una clase — se valida un comportamiento técnico puntual. En BDD, la unidad se llama **"característica" (feature)** — se valida si una funcionalidad completa se comporta como se espera desde la perspectiva del usuario, y el caso de prueba suele escribirse junto con product owners u otras partes no técnicas, no solo por el equipo de desarrollo.

TDD y BDD no son las únicas metodologías "pruebas primero" — también existen **SBE (Specification by Example)** y **ATDD (Acceptance Test-Driven Development)**, con el mismo espíritu de escribir el caso de uso antes que el código. TDD es la más enseñada y la más conocida de las metodologías "test-first" — aunque, como se aclara abajo, no es necesariamente la más practicada en su forma estricta en el día a día real de la industria.

## TDD en la teoría vs. en la práctica real

Vale la pena ser honesto acá: TDD en su forma estricta (escribir el test ANTES de cada línea de código, ciclo rojo-verde-refactor sostenido) **no es la práctica dominante en la industria real**, aunque los cursos y libros lo presenten como el estándar a seguir. Eso refleja el ideal que se enseña, no necesariamente lo que la mayoría de los equipos hace día a día.

Lo que sí es casi universal hoy, más allá de la etiqueta metodológica: tener una suite de tests automatizados corriendo en CI en cada Pull Request.

Lo más común en la práctica es **"test-after"**: se escribe el código, y después (a veces en el mismo PR, a veces como último paso antes de mergear) se escriben los tests para eso que ya se construyó. No es TDD estricto, pero tampoco es "no testear" — es el punto intermedio más frecuente en equipos reales.

**BDD con herramientas dedicadas** (Cucumber, SpecFlow) tiene una adopción bastante de nicho. Lo más común es escribir tests con nombres estilo Given/When/Then dentro de xUnit, sin el tooling completo de BDD — una versión liviana del concepto, no BDD "de libro".

**Por qué existe esta brecha:** TDD estricto exige una disciplina sostenida difícil de mantener bajo presión de fechas, y funciona mucho mejor cuando el problema ya está bien definido de entrada — es mucho más difícil de aplicar cuando todavía se está explorando cómo resolver algo.

**Lo que de verdad importa, más allá de la etiqueta metodológica:** que exista una suite de tests sólida cubriendo las reglas de negocio reales. El orden en que se escribió (antes o después del código) importa mucho menos en la práctica que el hecho de que el test exista y proteja algo real — la misma idea de "cobertura útil, no inflada" que se documenta más abajo en este archivo.

## Los 3 niveles de testing: qué prueba cada uno y cuándo corre

| Tipo | ¿Qué prueba? | ¿Cuándo se ejecuta? |
|---|---|---|
| **Unitarias (Unit Tests)** | Funciones aisladas o reglas de negocio específicas (ej. calcular un descuento, validar una regla), aislando dependencias con Mocks/Stubs. | Durante el desarrollo local, y automáticamente en cada commit/PR en CI. Rápidas y baratas de mantener. |
| **Integración (Integration Tests)** | La interacción real entre dos o más componentes (ej. que un repositorio guarde y lea bien de una base de prueba, o que un endpoint devuelva el JSON correcto). | Al completar un módulo/API, en CI o en staging, antes de fusionar a la rama principal. |
| **End-to-End (E2E)** | El flujo completo del usuario, desde la interfaz o API pública hasta el backend y la base de datos (ej: login → agregar al carrito → pagar). | Al finalizar funcionalidades críticas, o antes de un release mayor. Se automatizan con Cypress o Playwright — son pesadas y lentas, por eso suelen correr en pipelines nocturnos, no en cada commit. |

Este orden (unitarias → integración → E2E) se conoce como la **pirámide de testing**: a medida que se sube, aumenta el costo de recursos (tiempo, cómputo, dinero) y el alcance de lo que se prueba, pero conviene tener muchas menos pruebas — muchas unitarias en la base, bastantes menos de integración, y pocas E2E en la punta.

## Qué mockear en cada tipo de prueba

Mockear es crear una versión falsa de una dependencia para no depender de ella durante el test. El límite de qué se mockea cambia según el tipo de prueba:

- **En Unit Tests: se mockea TODO lo que no sea la unidad bajo prueba** — incluidas otras clases o servicios internos del propio sistema. El objetivo es aislar completamente la pieza que se está probando.
- **En Integration Tests: solo se mockean los servicios EXTERNOS** (una API de terceros, un proveedor de pagos). Las interacciones entre componentes INTERNOS del propio sistema (ej: el servicio llamando al repositorio real, contra una base de datos de prueba) deben correr de verdad — mockear de más en un test de integración le hace perder el sentido.

Confundir estos dos límites es un error común: si un test de integración mockea sus propios componentes internos, en la práctica es solo un test unitario más lento, no una prueba de integración real.

## Los 4 criterios de cobertura de código

"Cobertura de código" no es un solo número — hay distintos criterios para medirla, cada uno más exigente que el anterior:

- **Cobertura de funciones:** ¿se llamó a cada función al menos una vez?
- **Cobertura de sentencias:** ¿se ejecutó cada línea al menos una vez?
- **Cobertura de caminos:** ¿se ejecutó cada rama posible del flujo de control (cada combinación de if/else)?
- **Cobertura de condiciones:** ¿cada sub-expresión booleana se evaluó como verdadera Y como falsa?

Ejemplo con una función simple:

```csharp
public int? Sumar(int? x, int? y)
{
    if (x.HasValue && y.HasValue)
        return x + y;
    return null;
}
```

- Llamarla una vez (`Sumar(1, 2)`) da 100% de cobertura de **funciones**, pero deja sin ejecutar la rama del `else`.
- Agregar `Sumar(1, null)` cubre ambas ramas → 100% de cobertura de **sentencias** y **caminos**.
- Para 100% de cobertura de **condiciones**, hace falta que `x.HasValue` y `y.HasValue` se evalúen como true Y como false por separado: `Sumar(1, 2)`, `Sumar(1, null)`, `Sumar(null, 1)`, `Sumar(null, null)`.

**Con las 4 al 100%, ¿ya no hay bugs?** No. Estos 4 casos ni siquiera contemplan valores límite como `int.MaxValue + 1`, que en C# se desborda silenciosamente sin lanzar excepción a menos que el código esté en un bloque `checked`. El 100% de cobertura solo dice que el código SE EJECUTÓ, no que se probó con los valores correctos.

(La versión original de este ejemplo, en JavaScript, muestra que `numSum('1', '2')` devuelve `'12'` en vez de `3` por la conversión implícita de tipos del lenguaje. En C# el tipado estático evita ESE bug específico — no compilaría pasarle strings a parámetros `int?` — pero eso no significa que C# esté libre de sorpresas: el desborde silencioso de enteros es el equivalente real que sí aplica acá. Mismo principio que ya vimos en punto 1: verificar el comportamiento real de cada stack, no asumir que un bug de un lenguaje se traduce igual a otro.)

## Cobertura de código vs. cobertura de pruebas

Son dos métricas distintas, y se confunden seguido:

- **Cobertura de código:** % de líneas que se ejecutaron durante los tests.
- **Cobertura de pruebas:** % de funcionalidades o requerimientos (definidos en un documento de requisitos) que están cubiertos por al menos un test, sin importar cuántas líneas toque cada uno.

Un sistema puede tener 100% de cobertura de código y aun así tener baja cobertura de pruebas, si nunca se validó un requerimiento real del negocio. Ninguna de las dos garantiza ausencia de bugs — ambas ayudan a tener más confianza, no una garantía.

## Cobertura útil, no cobertura inflada

Un número de cobertura alto (ej. "95% de cobertura") no significa que el código esté libre de bugs — significa que el 95% de las líneas se ejecutaron al menos una vez durante los tests, nada más.

Es fácil inflar ese número testeando cosas que no tienen ningún riesgo real: un getter/setter automático, un DTO sin lógica. Eso sube el porcentaje sin agregar ninguna protección real.

**Cobertura útil** prioriza:
- Reglas de negocio (la lógica que, si se rompe, genera un problema real — como la validación de "no modificar pedidos completados").
- Casos borde (valores límite, entradas vacías o nulas, condiciones de carrera).
- Código que cambia seguido (cuanto más se toca un módulo, más protección necesita para no romperlo sin darse cuenta).

Y prioriza MENOS (o directamente no testea):
- Getters/setters automáticos sin lógica.
- Código generado automáticamente.
- Wrappers triviales que solo delegan a una librería externa sin agregar comportamiento propio.

## ¿Cuánto de cobertura hace falta en la práctica?

Nadie persigue el 100% en serio, salvo casos muy puntuales. Perseguirlo a cualquier costo lleva a la "cobertura inflada" de arriba, o peor, a lo que se conoce como **"coverage theater"**: tests que ejecutan una línea pero no verifican nada de verdad, solo para que la herramienta de cobertura la marque como cubierta. Eso da una falsa sensación de seguridad — es peor que no tener el test.

**Una referencia real, bastante citada en la industria** (viene de la guía interna de testing de Google, hecha pública):
- **60%** → aceptable
- **75%** → bueno
- **90%** → excelente

No es una ley, pero es mucho más realista que apuntar al 100%.

**Excepción real:** software regulado o safety-critical (aviónica, dispositivos médicos, el core de sistemas financieros) sí exige coberturas muy altas, a veces por certificaciones obligatorias. Para una API de negocio estándar, ese estándar no aplica.

**El patrón típico de un proyecto backend real** distribuye la cobertura de forma despareja, y eso es correcto, no un defecto:
- **Alta (cerca de 90-100%):** capa de dominio y aplicación — reglas de negocio, validadores, servicios. Es barata de testear y es donde más importa que no haya bugs.
- **Baja o nula:** DTOs, mapeos triviales, `Program.cs`, controllers que solo delegan — no agregan riesgo real.

El número total del proyecto es un promedio de ambas cosas — ver "68% de cobertura total" en un proyecto real y sano es normal, no una señal de alarma.

**Recomendación práctica:** no perseguir un número global.
1. Poner el foco en la capa de dominio/aplicación — ahí sí apuntar a 80-90%.
2. Usar el número como **piso de CI, no como objetivo a maximizar** (ej: "no se mergea si la cobertura de esa capa baja de 70%", no "hay que llegar a 95% como sea").
3. Herramientas como SonarQube (ver [04-deuda-tecnica.md](./04-deuda-tecnica.md)) permiten medir cobertura por proyecto/namespace — exigir 80% en Domain/Application y nada en Infrastructure/API, en vez de un número único para todo el sistema.
