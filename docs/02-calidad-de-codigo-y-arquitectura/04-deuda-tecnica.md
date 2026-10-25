# Deuda técnica

## En una frase

La deuda técnica es **el costo extra que pagas en cada cambio futuro por un atajo que tomaste hoy**; como una deuda financiera, genera intereses, así que hay que identificarla, cuantificarla en impacto de negocio y pagarla de a poco, no esconderla.

-----

## Antes de empezar

Conviene que ya sepas:

* Los principios SOLID y qué pasa cuando se violan: [SOLID y patrones](01-solid-y-patrones.md).
* Por qué refactorizar solo es seguro con pruebas: [Estrategia de testing](03-estrategia-de-testing.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Deuda técnica:** el costo futuro de un atajo tomado hoy.
* **Intereses (de la deuda):** el tiempo extra que cuesta cada cambio en el código afectado.
* **Code smell (olor de código):** señal superficial que sugiere un problema de diseño más profundo.
* **Número mágico:** valor fijo en el código sin un nombre que explique qué representa.
* **Big Ball of Mud (gran bola de barro):** sistema sin estructura reconocible, donde todo depende de todo.
* **Regla del Boy Scout:** dejar el código un poco mejor de lo que lo encontraste.
* **Análisis estático:** revisión automática del código sin ejecutarlo.
* **Complejidad ciclomática:** cantidad de caminos independientes que tiene un método; cuantos más, más difícil de entender y de probar.
* **Quality Gate:** condiciones de calidad que el código debe cumplir para poder fusionarse.

-----

## El problema

Hace dos años, para llegar a una fecha, el módulo de facturación se escribió así:

```csharp
public decimal Calcular(Factura f)
{
    decimal t = 0;
    foreach (var l in f.Lineas)
    {
        if (f.Tipo == 1) t += l.Precio * 1.18m;
        else if (f.Tipo == 2) t += l.Precio * 1.10m;
        else if (f.Tipo == 3 && l.Cantidad > 50) t += l.Precio * 0.95m;
        // ... 400 líneas más en el mismo método
    }
    return t;
}
```

Hoy, cada cambio en facturación tarda una semana: nadie sabe qué significa `Tipo == 3`, no hay pruebas y cada arreglo rompe otro caso. Cuando el equipo dice "hay que refactorizar facturación", producto responde: "no hay tiempo, hay features más importantes".

Las consecuencias:

* el atajo de hace dos años se sigue pagando, con intereses, en cada cambio;
* "el código está feo" no convence a nadie que decide prioridades;
* sin pruebas, refactorizar da miedo, así que la deuda crece en lugar de bajar.

-----

## Cómo funciona

### 1. Qué es la deuda técnica (y qué no)

Ward Cunningham acuñó la metáfora en 1992: entregar código no del todo correcto es como pedir un préstamo. Te permite ir más rápido hoy, pero cada día que no lo pagas, pagas intereses: cada cambio en esa zona cuesta más de lo que debería.

No toda deuda es mala. Martin Fowler la clasifica en un cuadrante:

| | Deliberada | Inadvertida |
| --- | --- | --- |
| **Imprudente** | "No hay tiempo para diseño" | "¿Qué es una arquitectura en capas?" |
| **Prudente** | "Lanzamos ya y lo pagamos en el próximo sprint" | "Ahora sabemos cómo deberíamos haberlo hecho" |

La deuda **prudente y deliberada** es una decisión de negocio válida, siempre que quede registrada y se pague. La peligrosa es la que nadie ve.

Deuda técnica no es solo código feo: no tener pruebas, logs, documentación ni dependencias actualizadas también es deuda, aunque el código se vea prolijo.

### 2. Paso 1: identificarla con code smells

Un **code smell** es una señal superficial de que puede haber un problema de diseño. No es un bug (el código funciona): es una pista de que esa parte merece revisión. El término lo acuñó Kent Beck y lo popularizó Martin Fowler en *Refactoring* (1999). Un olor no es un veredicto: invita a investigar, no confirma un problema por sí solo.

Code smells clásicos que ya aparecieron en este roadmap:

| Code smell | Qué es | Dónde se vio |
| --- | --- | --- |
| **Long Method / Large Class** (clase Dios) | Un método o clase que hace demasiado | [SOLID y patrones](01-solid-y-patrones.md): violación de SRP |
| **Switch Statements** | Cadenas de `if`/`switch` que crecen con cada caso nuevo | [SOLID y patrones](01-solid-y-patrones.md): violación de OCP |
| **Refused Bequest** | Una subclase hereda algo que no puede cumplir y lo rechaza con una excepción | [SOLID y patrones](01-solid-y-patrones.md): violación de LSP |
| **Duplicate Code** | La misma lógica repetida en varios lugares | [DRY, KISS y YAGNI](07-principios-de-simplicidad.md): DRY |
| **Speculative Generality** | Abstracciones "por si acaso" sin un caso real | [DRY, KISS y YAGNI](07-principios-de-simplicidad.md): YAGNI |
| **Long Parameter List** | Métodos con cinco, seis o más parámetros | Suele indicar que falta un objeto que los agrupe |

Otras señales de deuda: clases de 2000 líneas, consultas duplicadas sin optimizar, librerías desactualizadas y lógica de negocio mezclada con acceso a datos.

Un smell fácil de detectar son los **números mágicos**:

```csharp
// Número mágico: ¿qué significa "3"? Hay que buscar el contexto
if (pedido.Estado == 3)
{
    // ...
}
```

```csharp
// Con nombre: el código se explica solo
public enum EstadoPedido
{
    Pendiente = 1,
    Procesando = 2,
    Completado = 3,
    Cancelado = 4
}

if (pedido.Estado == EstadoPedido.Completado)
{
    // ...
}
```

El primero obliga a memorizar o buscar qué significa cada número. El segundo se lee solo. Lo mismo aplica a tasas, límites y umbrales: `1.18m` es un número mágico; `TasaIva` no.

### 3. A escala de sistema: Big Ball of Mud

Cuando la deuda se acumula durante años sin control, el resultado ya no es un smell en un método: es un sistema entero sin estructura. Brian Foote y Joseph Yoder lo llamaron **Big Ball of Mud** (1997): todo depende de todo, no hay límites entre módulos y cualquier cambio puede romper cualquier cosa.

```text
Code smells aislados  ──►  módulos acoplados  ──►  Big Ball of Mud
 (se arreglan con un        (se arreglan con un       (se arregla con una
  refactor puntual)          refactor planificado)     reescritura gradual)
```

Técnicamente no es un code smell sino un **antipatrón** de arquitectura, pero es a donde llevan los smells que nadie paga. Es el riesgo principal de un monolito sin disciplina (ver [Monolitos y monolito modular](../11-arquitectura-de-sistemas-y-estilos-arquitectonicos/03-monolitos-y-monolito-modular.md)).

### 4. Paso 2: cuantificarla y documentarla

"Este código está feo" no convence a nadie que decide prioridades. Hay que traducirlo a **impacto de negocio**:

| Queja informal | Ticket priorizable |
| --- | --- |
| "Facturación es un desastre" | "Cualquier cambio en facturación tarda 5 días en lugar de 1, porque no tiene pruebas y cada arreglo rompe otro caso. En el último trimestre fueron 4 incidentes en producción." |
| "Hay que actualizar las librerías" | "La versión de Newtonsoft.Json que usamos tiene una vulnerabilidad conocida (CVE-...) y bloquea la migración a .NET 10." |

Ese ticket va al backlog (Jira, Azure DevOps) como tarea técnica. Así deja de ser una queja y pasa a ser algo que se puede priorizar, estimar y planificar junto con las features.

### 5. Paso 3: pagarla con la Regla del Boy Scout

No hace falta esperar un mes dedicado a "limpieza", que casi nunca llega. La **Regla del Boy Scout** (popularizada por Robert C. Martin en *Clean Code*) dice: **deja el código un poco mejor de lo que lo encontraste**.

Si tienes que modificar un módulo para agregar una funcionalidad, aprovechas para refactorizar esa sección puntual, agregarle las pruebas que le faltan y dejarla más sólida, sin convertir la tarea en una reescritura completa que nadie pidió.

> **Ojo:** el refactor oportunista va en el mismo PR solo si es **chico y relacionado**. Si crece, sepáralo en otro PR: mezclar un refactor grande con una feature hace que el code review sea imposible.

### 6. Herramientas de análisis estático

Detectar deuda "a ojo" no escala. Estas herramientas la miden de forma automática:

| Herramienta | Qué hace |
| --- | --- |
| **SonarQube / SonarCloud** | Detecta code smells, bugs potenciales, vulnerabilidades y duplicación; calcula complejidad ciclomática y cobertura. Con **Quality Gates** puede bloquear un PR que no cumple un umbral mínimo |
| **Analizadores de Roslyn** (integrados en .NET) | Corren en el IDE y en `dotnet build`. Incluyen las reglas `CA` del SDK y analizadores de la comunidad como `StyleCop.Analyzers` |
| **`.editorconfig`** | Define reglas de estilo y su severidad para todo el equipo, versionadas en el repositorio |
| **`dotnet list package --outdated` / `--vulnerable`** | Lista dependencias desactualizadas o con vulnerabilidades conocidas |

La ventaja real: convierten "creo que este código es complicado" en un número concreto (complejidad ciclomática, porcentaje de duplicación, índice de mantenibilidad). Es justo lo que hace falta para el paso 2, en lugar de discutir percepciones.

Para que los analizadores fallen en CI y no queden como advertencias ignoradas:

```xml
<PropertyGroup>
  <AnalysisLevel>latest-recommended</AnalysisLevel>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

-----

## Ejemplo completo

El módulo de facturación del problema, con el ciclo completo.

**Paso 1: identificar.** SonarQube marca `Calcular` con complejidad ciclomática 47 (el umbral recomendado es 10-15), 0 % de cobertura y números mágicos (`1`, `2`, `3`, `1.18m`).

**Paso 2: cuantificar.** El ticket técnico:

```text
DEUDA-112: Facturación sin pruebas y con un método de 400 líneas

Impacto: los últimos 6 cambios en facturación tardaron en promedio 5 días
(estimados: 1 día) y generaron 4 incidentes en producción.
Propuesta: cubrir con pruebas los 3 tipos de factura y extraer el cálculo
por tipo. Se hace de forma incremental, junto con las features que toquen
facturación.
Estimación: +1 día en cada una de las próximas 3 features de facturación.
```

**Paso 3: Boy Scout.** Llega la feature "nuevo tipo de factura: exportación". En lugar de agregar un `else if` más:

1. Primero, pruebas que fijan el comportamiento actual (*characterization tests*), sin cambiar nada:

```csharp
[Theory]
[InlineData(TipoFactura.Nacional, 100, 118)]
[InlineData(TipoFactura.Reducida, 100, 110)]
public void Calcular_ComportamientoActual(TipoFactura tipo, decimal precio, decimal esperado)
{
    var factura = new Factura(tipo, [new Linea(precio, Cantidad: 1)]);

    Assert.Equal(esperado, new CalculadoraFactura().Calcular(factura));
}
```

2. Con las pruebas en verde, se reemplazan los números mágicos por nombres y se extrae el cálculo por tipo:

```csharp
public enum TipoFactura { Nacional = 1, Reducida = 2, Mayorista = 3, Exportacion = 4 }

public class CalculadoraFactura
{
    private const decimal TasaIva = 0.18m;
    private const decimal TasaReducida = 0.10m;
    private const decimal DescuentoMayorista = 0.05m;
    private const int CantidadMinimaMayorista = 50;

    public decimal Calcular(Factura factura) =>
        factura.Lineas.Sum(linea => CalcularLinea(factura.Tipo, linea));

    private static decimal CalcularLinea(TipoFactura tipo, Linea linea) => tipo switch
    {
        TipoFactura.Nacional => linea.Precio * (1 + TasaIva),
        TipoFactura.Reducida => linea.Precio * (1 + TasaReducida),
        TipoFactura.Mayorista when linea.Cantidad > CantidadMinimaMayorista => linea.Precio * (1 - DescuentoMayorista),
        TipoFactura.Exportacion => linea.Precio, // exenta de IVA
        _ => linea.Precio
    };
}
```

3. Se agrega la prueba de `Exportacion` y la feature queda terminada.

Qué observar:

* Las pruebas de caracterización van **antes** del refactor: fijan lo que el código hace hoy (incluso si es raro) para detectar cualquier cambio accidental.
* No se reescribió todo el módulo: se mejoró la parte que la feature necesitaba tocar.
* El `switch` se quedó: con 4 casos estables y en un solo lugar, es más simple que una Strategy por tipo. Si los tipos empezaran a crecer cada mes, ahí se justificaría.
* El ticket habló de días e incidentes, no de "código feo".

-----

## Errores comunes

**1. Pedir "un sprint de refactor".**
Qué pasa: nunca se aprueba, o se aprueba una vez y la deuda vuelve.
Por qué: compite contra features con valor visible y no tiene un impacto medido.
Arreglo: cuantifica el impacto y paga de forma incremental con la Regla del Boy Scout.

**2. Refactorizar sin pruebas.**
Qué pasa: el refactor introduce regresiones y el equipo concluye que "refactorizar es peligroso".
Por qué: no hay nada que avise cuando cambia el comportamiento.
Arreglo: primero pruebas de caracterización; después, refactor.

**3. Reescribir todo desde cero.**
Qué pasa: meses de trabajo, el sistema viejo sigue cambiando y la reescritura nunca alcanza al original.
Por qué: se subestima cuánto conocimiento del negocio vive en el código viejo.
Arreglo: reemplazo gradual, módulo por módulo (el patrón *Strangler Fig*).

**4. Tratar cada advertencia del analizador como deuda crítica.**
Qué pasa: miles de advertencias, el equipo las ignora todas.
Por qué: se activan todas las reglas de golpe, sin priorizar.
Arreglo: empieza con `latest-recommended`, corrige por severidad y haz que el código **nuevo** cumpla siempre (Quality Gate sobre el código nuevo, no sobre el total).

**5. Deuda deliberada sin registrar.**
Qué pasa: el "lo arreglamos después" se olvida.
Por qué: el atajo se tomó en una reunión y nadie lo anotó.
Arreglo: cada atajo deliberado genera un ticket en el momento en que se decide.

-----

## Cuándo sí y cuándo no

**Paga la deuda cuando:** el código se modifica seguido, genera incidentes o bloquea features. Ahí los intereses son altos.

**Toma deuda a propósito cuando:** validar una idea rápido vale más que la calidad (un MVP, una demo para un cliente) y queda registrado cuándo y cómo se va a pagar.

**No la pagues (todavía) cuando:** el código es feo pero estable y casi nunca se toca. Si no se modifica, no cobra intereses: refactorizarlo es costo sin retorno.

-----

## Resumen en 5 líneas

1. La deuda técnica es el costo futuro de un atajo de hoy; sus intereses son el tiempo extra de cada cambio.
2. Se identifica con code smells; acumulada durante años, termina en un Big Ball of Mud.
3. Se cuantifica en impacto de negocio (días, incidentes), no en "código feo".
4. Se paga de forma incremental con la Regla del Boy Scout, con pruebas antes de refactorizar.
5. SonarQube y los analizadores de Roslyn la miden y, con Quality Gates, evitan que crezca.

-----

## Para profundizar

<details>
<summary>El patrón Strangler Fig para sistemas legados</summary>

Martin Fowler lo nombró por una higuera que crece alrededor de un árbol hasta reemplazarlo. En lugar de reescribir un sistema de una vez, se pone una fachada delante (por ejemplo, un API Gateway) y se migran las funcionalidades una por una al sistema nuevo. Cada ruta migrada deja de llegar al sistema viejo, hasta que el sistema viejo se puede apagar. El riesgo de cada paso es chico y el sistema sigue funcionando todo el tiempo.

</details>

<details>
<summary>Dónde pagar primero: complejidad cruzada con frecuencia de cambio</summary>

No toda la deuda cobra los mismos intereses. Una técnica muy usada (Adam Tornhill, *Your Code as a Crime Scene*) cruza dos datos:

* **Complejidad:** de SonarQube o de un analizador.
* **Frecuencia de cambio:** de Git (`git log --format=format: --name-only | sort | uniq -c | sort -rn`).

Los archivos complejos **y** que cambian seguido (*hotspots*) son donde pagar la deuda rinde más. Un archivo complejo que no se toca en dos años puede esperar.

</details>

-----

## En entrevista

### Respuesta corta (junior)

La deuda técnica es lo que queda pendiente cuando se toman atajos para entregar más rápido, como no escribir pruebas o dejar código duplicado. Hay que arreglarla de a poco, por ejemplo mejorando el código cada vez que lo tocas.

### Respuesta ampliada (semi-senior)

La veo como una deuda financiera: a veces conviene tomarla, por ejemplo para validar un MVP, pero tiene que quedar registrada, porque cada cambio en ese código paga intereses. La identifico con code smells y con análisis estático, como SonarQube y los analizadores de Roslyn. Para priorizarla la traduzco a impacto de negocio: cuántos días extra cuesta cada cambio o cuántos incidentes genera. La pago de forma incremental con la Regla del Boy Scout, siempre con pruebas de caracterización antes del refactor, y priorizo los archivos complejos que además cambian seguido. Evito las reescrituras completas: prefiero migrar de a partes.

### Preguntas frecuentes de seguimiento

**1. ¿Cómo convences a producto de dedicar tiempo a la deuda técnica?**
Con números de negocio: cuánto tardan los cambios en esa zona comparado con lo estimado, cuántos incidentes generó y qué features bloquea. Y proponiendo pagarla de forma incremental junto con las features, no como un sprint aparte.

**2. ¿Toda deuda técnica es mala?**
No. La deuda deliberada y prudente ("lanzamos ya y lo pagamos después") es una decisión de negocio válida si queda registrada y se paga. La peligrosa es la inadvertida o la que nadie registra.

**3. ¿Qué es un code smell? Dame un ejemplo.**
Una señal de un posible problema de diseño, aunque el código funcione. Por ejemplo, un número mágico como `if (estado == 3)` o una clase de 2000 líneas con muchas responsabilidades.

-----

## Práctica

**Ejercicio 1.** Identifica al menos cuatro code smells en este código.

```csharp
public class Utilidades
{
    public decimal Procesar(int t, decimal m, bool f, string c, int d, bool e)
    {
        if (t == 1) m = m * 0.82m;
        if (t == 2) m = m * 0.82m - 5;
        if (f) m = m + 15;
        // if (e) m = m * 2;
        return m;
    }
}
```

<details>
<summary>Solución</summary>

* **Nombres sin significado:** `Utilidades`, `Procesar`, `t`, `m`, `f`.
* **Números mágicos:** `1`, `2`, `0.82m`, `5`, `15`.
* **Long Parameter List:** seis parámetros, y `c`, `d` y `e` ni siquiera se usan.
* **Duplicate Code:** `m * 0.82m` se repite.
* **Código comentado:** `// if (e) m = m * 2;` debería borrarse; Git lo recuerda.
* **Clase "Utilidades":** un nombre genérico que suele terminar acumulando responsabilidades sin relación.

</details>

**Ejercicio 2.** ¿Qué tiene de malo este ticket? Reescríbelo para que se pueda priorizar.

```text
TECH-45: Refactorizar el módulo de usuarios porque el código está muy mal.
```

<details>
<summary>Solución</summary>

* "Muy mal" no dice cuál es el problema ni cuánto cuesta.
* No hay impacto de negocio: nadie puede compararlo con una feature.
* "Refactorizar el módulo" no tiene alcance: puede ser un día o tres meses.

```text
TECH-45: Validaciones de usuarios duplicadas en 4 controllers

Impacto: la regla de "email único" está repetida en 4 lugares. En los
últimos 2 meses hubo 2 bugs por actualizar solo algunos de ellos.
Propuesta: mover la validación al servicio de usuarios y cubrirla con pruebas.
Alcance: 4 controllers, 1 servicio. Estimación: 1 día.
```

</details>

-----

## Siguiente lección

[Clase abstracta frente a interfaz](05-abstract-vs-interface.md)
