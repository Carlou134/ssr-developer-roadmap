# Deuda Técnica

## Definición

La deuda técnica es el costo que se paga cuando se toma un atajo en el código para entregar más rápido hoy, sacrificando calidad a largo plazo — igual que una deuda financiera, genera "intereses": cada cambio futuro en esa zona del código cuesta más tiempo del que debería.

## El ciclo de 3 pasos

### 1. Identificarla

**¿Qué es un code smell?** Es una señal superficial en el código que sugiere que puede haber un problema de diseño más profundo — no es un bug (el código sigue funcionando), es una pista de que ese código merece revisión. El término lo popularizó Martin Fowler en su libro *Refactoring* (atribuido originalmente a Kent Beck). Un "olor" no es un veredicto automático — es una heurística que invita a investigar, no confirma un problema por sí sola.

**Algunos code smells clásicos que ya aparecieron en este roadmap, con su nombre "oficial":**

| Code smell | Qué es | Dónde ya se vio |
|---|---|---|
| **Long Method / Large Class ("Clase Dios")** | Un método o clase que hace demasiado, difícil de seguir | [01-solid-y-patrones.md](./01-solid-y-patrones.md) — violación de SRP |
| **Switch Statements** | Cadenas de `if/else` o `switch` que crecen cada vez que se agrega un caso nuevo | [01-solid-y-patrones.md](./01-solid-y-patrones.md) — violación de OCP |
| **Refused Bequest** | Una subclase hereda algo que no puede cumplir, y lo rompe con una excepción | [01-solid-y-patrones.md](./01-solid-y-patrones.md) — violación de LSP |
| **Duplicate Code** | La misma lógica repetida en varios lugares | [07-principios-de-simplicidad.md](./07-principios-de-simplicidad.md) — DRY |
| **Speculative Generality** | Abstracciones armadas "por si acaso", sin un caso de uso real todavía | [07-principios-de-simplicidad.md](./07-principios-de-simplicidad.md) — YAGNI |

Detectar "code smells" o código frágil: una clase de 2000 líneas, consultas duplicadas a la base sin optimizar, librerías desactualizadas, métodos con demasiados parámetros, lógica de negocio mezclada con acceso a datos.

Un code smell clásico y fácil de detectar: los **números mágicos** — valores fijos usados directamente en el código, sin ningún nombre que explique qué representan.

```csharp
// Número mágico: ¿qué significa "3" acá? Hay que ir a buscar el contexto para saberlo.
if (pedido.Estado == 3)
{
    // ...
}

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

El primer caso obliga a memorizar o buscar qué significa cada número. El segundo se lee solo, sin necesitar contexto externo.

### 2. Cuantificarla y documentarla

No alcanza con decir "este código está feo". Hay que documentar el **impacto en el negocio**: *"Este módulo no tiene pruebas y tarda 3 días en modificarse en vez de 4 horas, porque cualquier cambio rompe otras partes."* Se registra como una tarea/ticket técnico en el backlog (Jira, Azure DevOps) — así deja de ser una queja informal y pasa a ser algo que se puede priorizar y planificar.

### 3. Acción — la Regla del Boy Scout

No hace falta esperar un mes entero dedicado a "limpieza". El enfoque práctico es: **dejar el código un poco mejor de lo que se encontró**. Si toca modificar un módulo para agregar una funcionalidad, se aprovecha para refactorizar esa sección puntual, agregarle los tests que le faltan, y dejarlo más sólido — sin convertir la tarea en una reescritura completa no pedida.

## Herramientas de análisis estático

Detectar deuda técnica "a ojo" no escala — estas herramientas la miden de forma objetiva y automática:

- **SonarQube / SonarCloud:** analiza el código en busca de code smells, bugs potenciales, vulnerabilidades de seguridad y código duplicado, y calcula métricas como la complejidad ciclomática. Se integra al pipeline de CI/CD con **Quality Gates** — se puede configurar para bloquear un Pull Request si no cumple un umbral mínimo de calidad, en vez de depender de que alguien lo note en el code review.
- **Roslyn Analyzers (integrados en .NET):** corren directo en el IDE y en tiempo de compilación (`dotnet build`), mostrando advertencias mientras se escribe el código. Incluyen los analizadores propios del framework (reglas `CA`) y otros de la comunidad como `StyleCop.Analyzers`, para forzar consistencia de estilo automáticamente.

La ventaja real de estas herramientas: convierten "creo que este código es complicado" en un número concreto (complejidad ciclomática, % de duplicación, índice de mantenibilidad) — eso es exactamente lo que hace falta para el paso 2 del ciclo (cuantificar), en vez de discutir sobre percepciones.
