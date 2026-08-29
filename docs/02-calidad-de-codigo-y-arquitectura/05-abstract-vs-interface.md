# Clase Abstracta vs. Interface, y Otras Decisiones de Diseño de Clases

## Clase abstracta vs. Interface

Ambas sirven para definir un contrato que otras clases deben cumplir, pero resuelven preguntas distintas.

**Clase abstracta:** una clase base que no se puede instanciar directamente, pensada como plantilla para clases que SON del mismo tipo — comparten propiedades y algo de lógica común.

```csharp
public abstract class Animal
{
    public string Nombre { get; set; } = string.Empty;

    public abstract void HacerSonido(); // obliga a las clases hijas a implementarlo
}

public class Perro : Animal
{
    public override void HacerSonido() => Console.WriteLine("Guau");
}
```

**Interface:** un contrato puro — sin implementación ni estado compartido, solo define qué métodos debe tener una clase. Sirve para definir CAPACIDADES, no un tipo compartido.

```csharp
public interface ICaminante
{
    void Caminar();
}

public class Robot : ICaminante
{
    public void Caminar() => Console.WriteLine("El robot camina");
}

public class Persona : ICaminante
{
    public void Caminar() => Console.WriteLine("La persona camina");
}
```

**La regla práctica para elegir:**

- Si la relación es "ES UN" (Perro ES UN Animal, comparte `Nombre` y lógica común) → clase abstracta.
- Si la relación es "PUEDE HACER" (un Robot y una Persona no son del mismo tipo, pero ambos PUEDEN caminar) → interface.

Un `Robot` y una `Persona` no tienen nada en común como tipo — no tiene sentido que ambos hereden de una clase base compartida. Pero sí comparten una capacidad (`Caminar`), y ahí es donde una interface tiene sentido.

## `sealed` vs `abstract`

Son casi opuestos en intención:

- **`abstract`:** "quiero que extiendas esto" — obliga a las clases hijas a implementar el comportamiento.
- **`sealed`:** "no quiero que nadie extienda esto" — bloquea la herencia por completo.

```csharp
public sealed class PagoService
{
    // Nadie puede hacer "class Otro : PagoService"
}
```

Se usa `sealed` cuando una clase ya está completa y no tiene sentido que se extienda — por ejemplo, para evitar que alguien sobrescriba comportamiento crítico de seguridad, o simplemente porque no hay ningún caso de uso real para heredarla. Conecta directo con OCP (ver [01-solid-y-patrones.md](./01-solid-y-patrones.md)): `sealed` es una declaración explícita de "esta clase está cerrada a extensión, a propósito".

## Sobrecarga (Overloading) vs. Sobrescritura (Override)

Se confunden seguido porque suenan parecido, pero son mecanismos distintos.

**Sobrecarga:** mismo nombre de método, distintos parámetros (distinta firma). Se resuelve en tiempo de compilación, según qué argumentos se pasan.

```csharp
public int Sumar(int a, int b) => a + b;
public double Sumar(double a, double b) => a + b;
```

**Sobrescritura:** una clase hija redefine el comportamiento de un método `virtual` (o `abstract`) heredado de la clase base — misma firma, comportamiento distinto.

```csharp
public class Animal
{
    public virtual void Sonido() => Console.WriteLine("Sonido genérico");
}

public class Perro : Animal
{
    public override void Sonido() => Console.WriteLine("Guau");
}
```

La diferencia clave: la sobrecarga es tener varias versiones del MISMO método en la MISMA clase; la sobrescritura es reemplazar el comportamiento de un método heredado en una clase HIJA.
