# Tokens y ventana de contexto

## En una frase

Un token es **la unidad mínima de texto que un modelo lee y escribe**; todo lo que haces con un LLM (lo que cabe, lo que cuesta y lo que tarda) se mide en tokens, no en palabras ni en caracteres.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es una API HTTP y un JSON: [Estándares RESTful](../03-diseno-de-apis-y-contratos/01-estandares-restful.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **LLM (*Large Language Model*):** modelo entrenado para predecir el siguiente token de un texto.
* **Token:** fragmento de texto (una palabra, parte de una palabra, un signo) con el que trabaja el modelo.
* **Tokenizador (*tokenizer*):** componente que convierte texto en tokens y tokens en texto.
* **Ventana de contexto (*context window*):** cantidad máxima de tokens que el modelo puede considerar en una sola llamada, sumando entrada y salida.
* **Tokens de entrada (*input tokens*):** todo lo que envías: prompt de sistema, historial, documentos, definiciones de herramientas.
* **Tokens de salida (*output tokens*):** lo que el modelo genera como respuesta.
* **Stateless:** sin memoria entre llamadas; cada petición debe incluir todo lo necesario.

-----

## El problema

Construyes un chat de soporte sobre un LLM. En desarrollo funciona perfecto. Para que "recuerde" la conversación, en cada mensaje reenvías el historial completo:

```csharp
// Cada vez que el usuario escribe, mandamos TODO el historial otra vez
history.Add(new Message("user", userInput));
var response = await llm.SendAsync(systemPrompt, history); // sin límite
history.Add(new Message("assistant", response.Text));
```

En producción aparecen los problemas:

* la factura crece mucho más rápido que la cantidad de mensajes;
* las conversaciones largas empiezan a fallar con un error de "prompt demasiado largo";
* las respuestas se vuelven lentas y el modelo ignora instrucciones que diste al principio;
* las respuestas a veces salen cortadas a la mitad.

Todo eso tiene la misma causa: no estás pensando en tokens.

-----

## Cómo funciona

### 1. Qué es un token

El modelo no ve letras ni palabras. El tokenizador parte el texto en piezas frecuentes del idioma con que se entrenó (normalmente con algoritmos como BPE, *Byte Pair Encoding*). Las palabras comunes suelen ser un token; las raras se parten en varias.

```text
Texto:   "Desarrollador semi-senior"
Tokens:  ["Des", "arroll", "ador", " semi", "-", "senior"]   ← ilustrativo
```

Reglas prácticas (aproximadas, dependen del modelo):

| Contenido | Cuántos tokens |
| --- | --- |
| Inglés | ~4 caracteres por token, ~0,75 palabras por token |
| Español | Algo más de tokens que el mismo texto en inglés |
| Código, JSON, UUIDs, Base64 | Muchos tokens: símbolos y cadenas raras se fragmentan |

> **Ojo:** cada familia de modelos tiene su propio tokenizador. El mismo texto da cantidades distintas en Claude, GPT o Llama. Si necesitas el número exacto, usa el endpoint o la librería de conteo del proveedor, no una regla de dedo.

### 2. El modelo genera de a un token

Un LLM produce la respuesta **token por token**: predice el siguiente, lo agrega y vuelve a predecir. Por eso:

* la latencia crece con la cantidad de tokens de salida;
* el *streaming* existe: te van llegando los tokens a medida que se generan;
* los tokens de salida suelen costar varias veces más que los de entrada.

### 3. La ventana de contexto

Es el tope de tokens que el modelo puede manejar en una llamada:

```text
ventana de contexto ≥ prompt de sistema + herramientas + historial + mensaje nuevo + respuesta
```

El modelo **no tiene memoria** entre llamadas (es *stateless*). Lo que parece memoria en un chat es que la aplicación reenvía el historial completo cada vez. Si te pasas del tope, la API rechaza la petición.

Si lo usas mal:

```csharp
// Historial sin recortar en una conversación de horas
var response = await llm.SendAsync(systemPrompt, history);
```

```text
400 invalid_request_error: prompt is too long: 215000 tokens > 200000 maximum
```

### 4. `max_tokens` y respuestas cortadas

`max_tokens` limita **solo la salida**. Si el modelo llega a ese límite, corta donde esté y la API te lo indica en el motivo de corte (`stop_reason: "max_tokens"` en la API de Anthropic, `finish_reason: "length"` en otras).

> **Ojo:** si no revisas el motivo de corte, vas a guardar o parsear respuestas incompletas, por ejemplo un JSON sin cerrar.

### 5. Costo

Los proveedores cobran por millón de tokens, separando entrada y salida:

```text
costo = (tokens_entrada × precio_entrada + tokens_salida × precio_salida) / 1.000.000
```

Como reenvías el historial, una conversación de N turnos no cuesta N veces un mensaje: cuesta aproximadamente **1 + 2 + 3 + … + N**, es decir, crece de forma cuadrática.

### 6. Más contexto no siempre es mejor

Aunque quepa, llenar la ventana tiene costo en calidad: el modelo presta menos atención a la información del medio de un contexto muy largo, y las instrucciones importantes se diluyen entre documentos irrelevantes. El contexto es un recurso escaso, como la memoria RAM: se administra.

-----

## Ejemplo completo

Resolvemos el chat de soporte: medimos tokens, ponemos un presupuesto y recortamos el historial.

`SupportChat/ConversationContext.cs`:

```csharp
public sealed class ConversationContext
{
    private const int ContextWindow = 200_000;
    private const int MaxOutputTokens = 1_024;
    private const int SafetyMargin = 2_000;

    private readonly List<Message> _history = new();
    private readonly ITokenCounter _counter;
    private readonly string _systemPrompt;

    public ConversationContext(string systemPrompt, ITokenCounter counter)
    {
        _systemPrompt = systemPrompt;
        _counter = counter;
    }

    public IReadOnlyList<Message> BuildRequest(string userInput)
    {
        _history.Add(new Message("user", userInput));

        var budget = ContextWindow - MaxOutputTokens - SafetyMargin
                     - _counter.Count(_systemPrompt);

        // Recorremos desde el mensaje más reciente hacia atrás
        var selected = new List<Message>();
        var used = 0;
        for (var i = _history.Count - 1; i >= 0; i--)
        {
            var tokens = _counter.Count(_history[i].Content);
            if (used + tokens > budget) break;
            selected.Insert(0, _history[i]);
            used += tokens;
        }
        return selected;
    }

    public void AddAssistantReply(LlmResponse response)
    {
        if (response.StopReason == "max_tokens")
            throw new IncompleteResponseException("La respuesta se cortó por max_tokens.");

        _history.Add(new Message("assistant", response.Text));
    }
}
```

Resultado: las conversaciones largas dejan de fallar, el costo por mensaje queda acotado y nunca guardas una respuesta truncada.

Qué observar:

* el presupuesto descuenta la salida (`MaxOutputTokens`), porque entrada y salida comparten la misma ventana;
* se conservan los mensajes **más recientes**, que son los que suelen importar; el prompt de sistema siempre se envía completo;
* `ITokenCounter` es una abstracción: en producción la implementas con el conteo oficial del proveedor, en tests con un contador falso;
* en un sistema real, en lugar de descartar mensajes viejos conviene **resumirlos** con el propio modelo y enviar el resumen.

-----

## Errores comunes

**1. Contar palabras en lugar de tokens.**
Qué pasa: los cálculos de costo y de límite fallan, sobre todo con código o con español.
Por qué: la relación palabra/token varía según el idioma y el contenido.
Arreglo: usa el contador oficial del proveedor para límites; las reglas de dedo, solo para estimar.

**2. Creer que el modelo recuerda.**
Qué pasa: "le dije en el mensaje anterior y no lo usó".
Por qué: el modelo es *stateless*; solo ve lo que viaja en la petición actual.
Arreglo: decide explícitamente qué historial y qué datos envías en cada llamada.

**3. Ignorar el motivo de corte.**
Qué pasa: JSON inválido o respuestas a medias guardadas en la base.
Por qué: el modelo llegó a `max_tokens` y cortó.
Arreglo: revisa siempre `stop_reason` y sube `max_tokens` o pide respuestas más cortas.

**4. Meter todo "por las dudas".**
Qué pasa: respuestas lentas, caras y peores.
Por qué: más tokens de entrada aumentan costo y latencia, y diluyen lo importante.
Arreglo: envía solo lo relevante (por ejemplo, con búsqueda/RAG) y usa *prompt caching* para la parte fija.

-----

## Evolución y versiones (opcional)

* **2020-2022:** ventanas de pocos miles de tokens (GPT-3 tenía 2.048, luego 4.096).
* **2023-2024:** ventanas de 100.000 a 200.000 tokens se vuelven comunes; aparece *prompt caching*.
* **2025 en adelante:** ventanas de hasta un millón de tokens en algunos modelos. El problema pasa de "no entra" a "entra, pero es caro y se diluye".

-----

## Cuándo sí y cuándo no

**Mide tokens cuando:** integras un LLM en un producto, estimas costos, procesas documentos largos o mantienes conversaciones de varios turnos.

**No te obsesiones cuando:** usas un asistente de forma personal y el costo es fijo. Igual vale saber que un contexto cargado responde peor.

-----

## Resumen en 5 líneas

1. Un token es un fragmento de texto; el modelo lee y escribe en tokens.
2. La ventana de contexto limita entrada + salida en una sola llamada.
3. El modelo no tiene memoria: la aplicación reenvía el historial, y eso se paga cada vez.
4. Se cobra por token de entrada y de salida; la salida es más cara y más lenta.
5. El contexto es un recurso escaso: mándale lo relevante, no todo.

-----

## Para profundizar (opcional)

<details>
<summary>¿Qué es prompt caching y por qué abarata tanto?</summary>

Si el inicio de tus peticiones es siempre igual (prompt de sistema, definiciones de herramientas, un documento grande), el proveedor puede guardar el procesamiento de ese prefijo. Las siguientes llamadas que comparten el prefijo leen de la caché a una fracción del precio normal y con menos latencia. Por eso conviene poner lo fijo **al principio** y lo variable **al final** del prompt.

</details>

<details>
<summary>¿Por qué los tokens de salida son más caros?</summary>

La entrada se procesa en paralelo en una sola pasada. La salida se genera de a un token, y cada token nuevo requiere otra pasada del modelo. Es más trabajo de cómputo por token.

</details>

-----

## En entrevista (opcional)

### Respuesta corta (junior)

Un token es la unidad de texto con la que trabaja un modelo; puede ser una palabra o parte de ella. Los modelos tienen un límite de tokens por llamada, la ventana de contexto, y se cobra por tokens de entrada y de salida.

### Respuesta ampliada (semi-senior)

Los tokens definen tres cosas: lo que cabe (ventana de contexto, que incluye entrada y salida), lo que cuesta (precio por millón, con la salida más cara) y lo que tarda (la salida se genera token por token). Como el modelo es *stateless*, en un chat reenvías el historial y el costo crece de forma cuadrática con los turnos, así que hay que recortar o resumir. Además, llenar la ventana degrada la calidad, por eso uso RAG para traer solo lo relevante y *prompt caching* para el prefijo fijo. Y siempre reviso el `stop_reason` para no procesar respuestas truncadas.

### Preguntas frecuentes de seguimiento

**1. ¿Una ventana de 1 millón de tokens elimina la necesidad de RAG?**
No. Cabe más, pero cada llamada cuesta y tarda más, y la precisión cae con contextos muy largos. RAG sigue siendo más barato y suele ser más preciso.

**2. ¿Qué pasa si la respuesta supera `max_tokens`?**
Se corta en ese punto y el motivo de corte lo indica. Hay que detectarlo y reintentar o pedir continuación.

**3. ¿Por qué un JSON grande consume tantos tokens?**
Llaves, comillas, nombres de campos repetidos e identificadores largos se tokenizan mal. Un formato más compacto o enviar solo los campos necesarios reduce mucho el consumo.

-----

## Práctica

**Ejercicio 1.** Un chat envía un prompt de sistema de 1.000 tokens y cada turno agrega 200 tokens de usuario y 300 de respuesta. Sin recortar el historial, ¿cuántos tokens de entrada se envían en el turno 10?

<details>
<summary>Solución</summary>

En el turno 10 viajan el sistema (1.000), los 9 turnos anteriores completos (9 × 500 = 4.500) y el mensaje nuevo (200): **5.700 tokens de entrada**. En el turno 1 eran 1.200. Sumando los 10 turnos, la entrada total es 1.200 + 1.700 + … + 5.700 = 34.500 tokens, mucho más que los 5.000 de texto "nuevo" que se intercambiaron.

</details>

**Ejercicio 2.** ¿Qué tiene de malo esto?

```csharp
var response = await llm.SendAsync(prompt, maxTokens: 200);
var order = JsonSerializer.Deserialize<Order>(response.Text);
```

<details>
<summary>Solución</summary>

Si la respuesta necesita más de 200 tokens, se corta y el JSON queda incompleto: `Deserialize` lanza una excepción o, peor, deserializa parcialmente. Hay que revisar `response.StopReason` antes de parsear y dimensionar `maxTokens` según el tamaño esperado de la salida.

</details>

-----

## Siguiente lección

[MCP (Model Context Protocol)](02-mcp-model-context-protocol.md)
