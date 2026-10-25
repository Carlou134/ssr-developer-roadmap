# DRY, KISS y YAGNI

## En una frase

Tres principios que atacan la complejidad innecesaria desde ángulos distintos: **DRY**, no repetir el mismo conocimiento; **KISS**, resolver con la solución más simple que funcione; **YAGNI**, no construir hoy lo que nadie pidió. Y la madurez está en saber cuándo el código simple empieza a doler y recién ahí refactorizar hacia SOLID.

-----

## Antes de empezar

Conviene que ya sepas:

* Por qué la complejidad es el primer trade-off: [¿Qué son los trade-offs?](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md).
* Cuándo se justifica una abstracción: [SOLID y patrones](01-solid-y-patrones.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **DRY (*Don't Repeat Yourself*):** cada pieza de conocimiento debe tener una única representación en el sistema.
* **KISS (*Keep It Simple, Stupid*):** la solución más simple que funcione suele ser la mejor.
* **YAGNI (*You Aren't Gonna Need It*):** no implementes algo hasta que realmente lo necesites.
* **Abstracción incorrecta:** código compartido que une casos que en realidad cambian por razones distintas.
* **Regla de tres:** extraer una abstracción recién cuando el mismo código aparece en tres lugares.
* **Sobrearquitectura:** una solución más grande o compleja de lo que el problema necesita.
* **Flag booleano (parámetro bandera):** parámetro `bool` que cambia el comportamiento de un método, como `esContratista`.

-----

## El problema

Un equipo nota que el cálculo del pago de empleados de planta y el de contratistas comparten tres líneas. Alguien los unifica:

```csharp
public decimal CalcularPago(Persona p, bool esContratista, bool incluyeBono, bool esFinDeMes, bool aplicaRetencion)
{
    decimal pago = p.HorasTrabajadas * p.TarifaHora;

    if (!esContratista && incluyeBono) pago += p.Bono;
    if (esContratista && esFinDeMes) pago -= 50;
    if (aplicaRetencion && !esContratista) pago *= 0.87m;
    if (aplicaRetencion && esContratista) pago *= 0.92m;
    // ... 12 condiciones más agregadas en los últimos 8 meses

    return pago;
}
```

Al mismo tiempo, en otro módulo, la regla de "descuento de 10 % a clientes VIP" está copiada en cinco lugares distintos. Cuando cambia al 12 %, se actualizan cuatro.

Y en un tercer módulo, alguien construyó un motor de reportes configurable con exportación a Excel, PDF y CSV, "porque seguro lo van a pedir". Un año después, solo se usa un reporte en CSV.

Las consecuencias:

* el cálculo de pagos es una **abstracción incorrecta**: cualquier cambio en la regla de contratistas arriesga romper la de empleados de planta;
* el descuento VIP es **duplicación real** de conocimiento: un cambio de negocio dejó al sistema inconsistente;
* el motor de reportes es complejidad que se mantiene sin que nadie la use.

-----

## Cómo funciona

### 1. DRY: no repetir el mismo conocimiento

Andrew Hunt y David Thomas lo definieron en *The Pragmatic Programmer* (1999): **"cada pieza de conocimiento debe tener una representación única, inequívoca y autorizada dentro del sistema"**.

Si una regla de negocio (como un descuento) está copiada en tres archivos, el día que cambie hay que actualizarla en los tres, y es cuestión de tiempo hasta que alguien se olvide de uno.

La palabra clave es **conocimiento**, no código. DRY no pide eliminar toda repetición: dos fragmentos que hoy se ven iguales pero representan conceptos distintos del negocio pueden divergir mañana. DRY aplica a lo que representa el **mismo** conocimiento, no a coincidencias superficiales.

### 2. DRY frente a la abstracción incorrecta

Sandi Metz lo resumió en una frase: **"la duplicación es mucho más barata que la abstracción incorrecta"**.

Unir antes de tiempo dos procesos que comparten tres líneas puede terminar en una función gigante llena de flags booleanos (`esContratista`, `esEdicion`, `esVenta`) que intenta acomodar casos que en el fondo son distintos. Cuando uno de los procesos cambia por una razón que no afecta al otro, la abstracción compartida se vuelve un cuello de botella: cualquier cambio arriesga romper algo que no tenía nada que ver.

| Pregunta equivocada | Pregunta correcta |
| --- | --- |
| ¿Se ven parecidos? | ¿**Cambian por la misma razón**? |

Si dos cálculos hoy hacen lo mismo pero responden a reglas de negocio independientes (el pago de un empleado de planta y el de un contratista, regidos por normas distintas), van separados aunque compartan una línea.

**Regla de tres** (Don Roberts, popularizada por Martin Fowler en *Refactoring*): no extraigas código repetido la primera vez que lo ves duplicado, ni la segunda. Espera a verlo en **al menos tres lugares**. Recién ahí se ve cómo varía de verdad, y puedes diseñar la abstracción correcta en lugar de adivinarla con dos casos.

> **Ojo:** la regla de tres es para código que **se parece**. Si es exactamente la misma regla de negocio (el descuento VIP del problema), centralízala desde la segunda copia: ahí ya sabes que es el mismo conocimiento.

### 3. KISS: la solución más simple que funcione

El principio viene de la ingeniería aeronáutica (Kelly Johnson, Lockheed, década de 1960): diseñar para que cualquier mecánico pudiera repararlo con herramientas básicas. También se dice "Keep It Short and Simple", una variante más amable.

```csharp
// Sobrecomplicado
if (user != null && user.IsActive == true)
{
    // ...
}

// Simple, mismo resultado
if (user?.IsActive == true)
{
    // ...
}
```

(El `== true` es necesario: `user?.IsActive` es un `bool?`, que puede ser `null`).

La violación típica de KISS es la **sobrearquitectura**: diseñar una solución demasiado grande para un problema simple, como microservicios para una aplicación con cien usuarios. Se conecta con [simplicidad frente a flexibilidad](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md).

### 4. Legibilidad frente a brevedad

Simple no es lo mismo que corto. Comprimir todo en una línea no lo hace más simple:

```csharp
// Compacto: hay que leerlo de un tirón para entenderlo
public decimal CalcularValorInventario(List<Producto> inventario) =>
    inventario.Where(p => p.Activo && p.Stock > 50).Sum(p => (p.Precio * 0.90m) * p.Stock);
```

```csharp
// Desglosado en pasos con nombres
public decimal CalcularValorInventario(List<Producto> inventario)
{
    const int StockMinimoParaValuar = 50;
    const decimal FactorDescuento = 0.90m;

    var productosAptos = inventario.Where(p => p.Activo && p.Stock > StockMinimoParaValuar);

    decimal valorTotal = 0m;
    foreach (var producto in productosAptos)
    {
        decimal precioConDescuento = producto.Precio * FactorDescuento;
        valorTotal += precioConDescuento * producto.Stock;
    }

    return valorTotal;
}
```

El código se lee muchas más veces de las que se escribe. Con la segunda versión, si algo calcula mal, pones un breakpoint en `precioConDescuento` y ves el valor de cada producto. En la versión de una línea también se puede depurar (Visual Studio y Rider permiten breakpoints dentro de una lambda), pero es bastante más incómodo.

No es una regla absoluta: una cadena LINQ corta y con nombres claros puede ser **más** legible que un `foreach`. La pregunta es: ¿alguien que no escribió esto lo entiende en una lectura?

### 5. YAGNI: no construyas lo que nadie pidió

Viene de Extreme Programming (Ron Jeffries): **implementa las cosas cuando realmente las necesitas, nunca cuando solo prevés que las vas a necesitar**.

Ejemplo de violación: el cliente pidió "guardar usuarios", y se termina construyendo cinco medios de pago, un sistema de reportes y exportación a Excel. Nada fue pedido, y buena parte probablemente nunca se use.

Lo que construyes "por si acaso" no es gratis:

| Costo | Por qué |
| --- | --- |
| Construirlo | Tiempo que no se usó en lo que sí se pidió |
| Mantenerlo | Hay que actualizarlo, probarlo y entenderlo en cada cambio |
| Equivocarse | Cuando el requerimiento real llega, suele ser distinto de lo imaginado, y hay que rehacerlo |

Es el mismo principio que **No Big Design Up Front** de [los trade-offs](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md): la mitad de lo que se anticipa nunca ocurre, y la otra mitad termina siendo distinta de como se imaginó.

> **Ojo:** YAGNI no aplica a lo que es caro agregar después y es casi seguro que hará falta: seguridad, logs, pruebas, una estructura de proyecto razonable. YAGNI es sobre **funcionalidades especulativas**, no sobre calidad básica.

### 6. La diferencia entre los tres

| Principio | Te protege de... | Pregunta |
| --- | --- | --- |
| **DRY** | Conocimiento duplicado que queda inconsistente | ¿Esta regla ya existe en otro lugar? |
| **KISS** | Soluciones más complejas de lo necesario | ¿Hay una forma más simple que funcione igual? |
| **YAGNI** | Funcionalidades que nadie pidió | ¿Alguien necesita esto **hoy**? |

Los tres apuntan a mantener baja la complejidad del sistema: el primer trade-off de [la fase 1](../01-autonomia-resolucion-problemas/05-que-son-los-tradeoffs.md).

### 7. ¿Cambian en otros lenguajes?

A diferencia de SOLID y de los patrones, **DRY, KISS y YAGNI no dependen de que el lenguaje sea orientado a objetos**. Son sobre cómo manejar la complejidad y el conocimiento duplicado, algo universal en TypeScript, Python, Java o cualquier paradigma. Aquí no hay diferencias de comportamiento que verificar: la idea se traslada uno a uno.

### 8. Cuándo pasar de KISS a SOLID

KISS y SOLID no compiten: son dos momentos distintos del mismo código. La pregunta no es "¿KISS o SOLID?", sino **"¿en qué momento dejo de aplicar KISS y refactorizo hacia SOLID?"**.

La respuesta: **cuando el código simple empieza a doler**. Se escribe simple mientras cumple lo que se pide (KISS + YAGNI). Se refactoriza hacia SOLID cuando un requerimiento real vuelve ese código difícil de manejar: ni un día antes, "por las dudas", ni mucho después, con el dolor ya instalado.

```text
Código simple ──► llega un cambio real ──► ¿duele? ──No──► sigue simple
                                              │
                                             Sí
                                              │
                                              ▼
                                   refactor hacia SOLID
                                   (con pruebas antes)
```

Es la Regla del Boy Scout de [Deuda técnica](04-deuda-tecnica.md) aplicada al diseño, y el último paso del framework de trade-offs: empezar simple, optimizar después.

La madurez técnica no se mide por usar patrones desde el día uno, sino por reconocer el momento en que hacen falta.

-----

## Ejemplo completo

Los tres casos del problema, resueltos.

**1. Pago: separar lo que cambia por razones distintas.** Se deshace la abstracción incorrecta. Las tres líneas comunes se duplican a propósito:

```csharp
public class CalculadoraPagoPlanta
{
    private const decimal RetencionPlanta = 0.13m;

    public decimal Calcular(EmpleadoPlanta e)
    {
        decimal pago = e.HorasTrabajadas * e.TarifaHora + e.Bono;
        return pago * (1 - RetencionPlanta);
    }
}

public class CalculadoraPagoContratista
{
    private const decimal RetencionContratista = 0.08m;
    private const decimal CargoAdministrativoMensual = 50m;

    public decimal Calcular(Contratista c, bool esFinDeMes)
    {
        decimal pago = c.HorasTrabajadas * c.TarifaHora;
        if (esFinDeMes) pago -= CargoAdministrativoMensual;
        return pago * (1 - RetencionContratista);
    }
}
```

Ahora un cambio en la ley de contratistas solo toca `CalculadoraPagoContratista`, y sin flags booleanos.

**2. Descuento VIP: centralizar el mismo conocimiento.**

```csharp
public static class PoliticaComercial
{
    public const decimal DescuentoVip = 0.12m;

    public static decimal AplicarDescuentoVip(decimal monto) => monto * (1 - DescuentoVip);
}
```

Los cinco lugares llaman a `PoliticaComercial.AplicarDescuentoVip`. El próximo cambio de porcentaje se hace en una línea.

**3. Reportes: borrar lo que no se usa (YAGNI).** Se elimina el motor configurable y las exportaciones a Excel y PDF. Queda una clase que genera el único reporte CSV que se usa. Si mañana piden PDF, se agrega **ese** requerimiento, con sus detalles reales.

Qué observar:

* En el caso 1 se **duplicó** código, y fue la decisión correcta: las reglas cambian por razones distintas.
* En el caso 2 se **centralizó**, y también fue correcto: es la misma regla de negocio.
* La diferencia entre los dos no está en el código, sino en la pregunta "¿cambian por la misma razón?".
* Borrar código (caso 3) también es diseñar: el código que no existe no tiene bugs ni hay que mantenerlo.

-----

## Errores comunes

**1. Aplicar DRY a coincidencias de código.**
Qué pasa: funciones compartidas llenas de flags booleanos que nadie se anima a tocar.
Por qué: se unifica lo que se ve igual, no lo que cambia por la misma razón.
Arreglo: pregunta "¿cambian por la misma razón?"; si no, duplica a propósito.

**2. No aplicar DRY a reglas de negocio.**
Qué pasa: la misma regla con valores distintos en distintas partes del sistema.
Por qué: se copia la regla "porque es una sola línea".
Arreglo: cada regla de negocio en un solo lugar, con un nombre.

**3. Confundir simple con corto.**
Qué pasa: una línea de 200 caracteres que nadie entiende en una lectura.
Por qué: se mide la simplicidad en líneas, no en esfuerzo de comprensión.
Arreglo: pasos intermedios con nombres que expliquen la intención.

**4. Usar YAGNI como excusa para no hacer calidad.**
Qué pasa: sin pruebas, sin logs y sin validaciones "porque no se pidieron".
Por qué: se aplica YAGNI a la calidad, no a las funcionalidades.
Arreglo: YAGNI es para funcionalidades especulativas; seguridad, pruebas y observabilidad no son opcionales.

**5. Esperar demasiado para refactorizar.**
Qué pasa: un método de 400 líneas que "todavía funciona".
Por qué: KISS se interpreta como "nunca refactorizar".
Arreglo: cuando el cambio duele, es el momento: refactoriza con pruebas antes.

-----

## Cuándo sí y cuándo no

**Aplica DRY cuando:** es la misma regla de negocio o el mismo conocimiento, aunque esté en solo dos lugares.

**No apliques DRY cuando:** el código se parece pero responde a reglas que cambian por razones distintas, o solo lo viste repetido una o dos veces.

**Aplica KISS y YAGNI siempre como punto de partida.** Sal de ellos cuando aparece un dolor real: acoplamiento, cambios frecuentes confirmados o necesidad de aislar para probar.

-----

## Resumen en 5 líneas

1. DRY: cada conocimiento en un solo lugar; aplica a reglas de negocio, no a coincidencias de código.
2. "La duplicación es más barata que la abstracción incorrecta": unifica solo lo que cambia por la misma razón.
3. KISS: la solución más simple que funcione; simple no es lo mismo que corto.
4. YAGNI: no construyas funcionalidades que nadie pidió, pero no lo uses como excusa para omitir la calidad.
5. Escribe simple y refactoriza hacia SOLID cuando el código empieza a doler, no antes.

-----

## Para profundizar

<details>
<summary>WET y AHA: los nombres de la tensión</summary>

* **WET** (*Write Everything Twice*): la versión burlona del código que viola DRY. A veces se usa en serio como "está bien escribirlo dos veces antes de abstraer".
* **AHA** (*Avoid Hasty Abstractions*, Kent C. Dodds): prefiere la duplicación hasta que el cambio te muestre la abstracción correcta. Es la regla de tres con otro nombre.

Todos dicen lo mismo: la abstracción se descubre con casos reales, no se inventa con uno solo.

</details>

<details>
<summary>Cómo deshacer una abstracción incorrecta</summary>

Sandi Metz propone un proceso concreto cuando ya tienes una abstracción llena de flags:

1. Vuelve a poner el código de la abstracción dentro de cada lugar que la llama (*inline*).
2. En cada lugar, borra las ramas que no aplican a ese caso.
3. Ahora tienes código duplicado, pero cada copia es simple y solo hace lo suyo.
4. Si con el tiempo aparece una abstracción real, extráela de nuevo, esta vez con evidencia.

Da miedo porque "agrega" duplicación, pero quita el acoplamiento entre casos que no tenían nada que ver.

</details>

-----

## En entrevista

### Respuesta corta (junior)

DRY es no repetir código, KISS es mantener las soluciones simples y YAGNI es no programar cosas que todavía no se necesitan. Los tres sirven para que el código sea más fácil de mantener.

### Respuesta ampliada (semi-senior)

DRY no es "no repetir código", sino no repetir **conocimiento**: una regla de negocio debe vivir en un solo lugar. Pero si dos fragmentos se parecen y cambian por razones distintas, prefiero duplicarlos, porque la abstracción incorrecta, llena de flags, es más cara que la duplicación. Para abstraer código similar espero la regla de tres. KISS para mí es optimizar la comprensión, no la cantidad de líneas. Aplico YAGNI a las funcionalidades especulativas, no a la calidad: seguridad, logs y pruebas van siempre. Empiezo simple y refactorizo hacia SOLID cuando un cambio real empieza a doler, con pruebas antes del refactor.

### Preguntas frecuentes de seguimiento

**1. ¿Cuándo duplicarías código a propósito?**
Cuando dos fragmentos se parecen pero responden a reglas que cambian por razones distintas. Unirlos crearía acoplamiento entre casos que no tienen nada que ver.

**2. ¿YAGNI significa no pensar en el futuro?**
No. Significa no **implementar** lo que no se pidió. Pensar en el futuro está bien, por ejemplo para no tomar decisiones difíciles de revertir; lo que se evita es construir funcionalidades especulativas.

**3. ¿Qué es la regla de tres?**
Esperar a ver el mismo código en tres lugares antes de extraer una abstracción, para diseñarla con evidencia de cómo varía y no adivinarla con dos casos.

-----

## Práctica

**Ejercicio 1.** ¿Unificarías estos dos métodos? ¿Por qué?

```csharp
public decimal CalcularEnvioNacional(decimal peso) => peso * 5m + 10m;
public decimal CalcularEnvioInternacional(decimal peso) => peso * 5m + 10m;
```

Contexto: el envío nacional lo define el área de logística; el internacional depende de acuerdos con aduanas que se renegocian cada año.

<details>
<summary>Solución</summary>

**No.** Hoy coinciden, pero cambian por razones distintas: logística por un lado, acuerdos de aduana por otro. Unificarlos haría que la próxima renegociación internacional obligue a agregar un flag `esInternacional` en una función compartida. Mantenerlos separados es la decisión correcta, aunque el código sea idéntico hoy. Sí convendría darles nombres a las constantes (`TarifaPorKgNacional`, `CargoFijoInternacional`) en cada uno.

</details>

**Ejercicio 2.** ¿Qué principio viola esta clase? ¿Qué harías?

```csharp
public class ExportadorUsuarios
{
    public byte[] Exportar(List<Usuario> usuarios, string formato, bool comprimir,
        bool cifrar, string idioma, bool incluirEncabezados, char separador)
    {
        // soporta CSV, Excel, PDF, XML y JSON
        // ...
    }
}
```

Contexto: el único uso en todo el sistema es `Exportar(usuarios, "csv", false, false, "es", true, ',')`.

<details>
<summary>Solución</summary>

**YAGNI** (y, de paso, KISS): soporta cinco formatos, compresión, cifrado e idiomas, y solo se usa una combinación. Todo lo demás es código que hay que mantener sin que nadie lo use.

```csharp
public class ExportadorUsuariosCsv
{
    public byte[] Exportar(IEnumerable<Usuario> usuarios)
    {
        var csv = new StringBuilder("Nombre,Email\n");
        foreach (var u in usuarios) csv.AppendLine($"{u.Nombre},{u.Email}");
        return Encoding.UTF8.GetBytes(csv.ToString());
    }
}
```

Si mañana piden Excel, se agrega con los requisitos reales. Y si aparecen varios formatos de verdad, ahí se justifica una interfaz `IExportador` (Strategy).

</details>

-----

## Siguiente lección

[Arquitectura en capas (N-Tier) en la práctica](08-arquitectura-en-capas-ntier.md)
