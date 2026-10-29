# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Agente (*agent*).** LLM que, en un bucle, decide qué herramienta usar, ve el resultado y sigue hasta terminar la tarea.

**Carga progresiva (*progressive disclosure*).** Cargar la información en niveles, de lo mínimo a lo completo, solo cuando hace falta. Es la base de las skills.

**Cliente MCP.** Componente dentro del host que mantiene una conexión 1 a 1 con un servidor MCP.

**Conocimiento procedimental.** Saber **cómo** hacer algo (pasos, convenciones, criterios), no solo **qué** es.

**Frontmatter.** Bloque YAML al inicio de un archivo Markdown, entre líneas `---`. En `SKILL.md` contiene `name` y `description`.

**Host.** La aplicación de IA que usa la persona (Claude Desktop, Claude Code, un IDE). Contiene el modelo y los clientes MCP.

**JSON-RPC 2.0.** Formato de mensajes petición/respuesta en JSON. Es el que usa MCP.

**LLM (*Large Language Model*).** Modelo entrenado para predecir el siguiente token de un texto.

**`max_tokens`.** Límite de tokens de salida de una llamada. Si se alcanza, la respuesta se corta.

**MCP (*Model Context Protocol*).** Protocolo abierto para conectar aplicaciones de IA con herramientas y datos externos de forma estándar.

**Prompt caching.** Reutilizar el procesamiento de un prefijo de prompt que se repite entre llamadas, para pagar menos y responder más rápido.

**Prompt de sistema (*system prompt*).** Instrucciones que acompañan siempre a la conversación.

**Prompt injection.** Texto malicioso dentro de los datos (un documento, un resultado de herramienta) que intenta darle órdenes al modelo.

**RAG (*Retrieval-Augmented Generation*).** Buscar solo los fragmentos relevantes de una base de conocimiento y enviarlos al modelo junto con la pregunta.

**Resource (MCP).** Dato de solo lectura, identificado por una URI, que la aplicación adjunta al contexto.

**Servidor MCP.** Programa que expone tools, resources y prompts siguiendo el protocolo MCP.

**Skill (*Agent Skill*).** Carpeta con un `SKILL.md`, scripts y recursos que enseña a un agente a hacer una tarea concreta y se carga bajo demanda.

**Stateless.** Sin memoria entre llamadas. Un LLM lo es: cada petición debe incluir todo lo necesario.

**stdio.** Transporte MCP en el que el host lanza el servidor como proceso local y se comunican por entrada y salida estándar.

**Stop reason.** Motivo por el que el modelo dejó de generar: terminó, llegó a `max_tokens` o pidió usar una herramienta.

**Streamable HTTP.** Transporte MCP para servidores remotos sobre HTTP, con *streaming* opcional.

**Token.** Fragmento de texto (una palabra, parte de una palabra, un signo) con el que trabaja el modelo. Es la unidad de costo y de límite.

**Tokenizador (*tokenizer*).** Componente que convierte texto en tokens y tokens en texto. Cada familia de modelos tiene el suyo.

**Tokens de entrada (*input tokens*).** Todo lo que envías: prompt de sistema, historial, documentos y definiciones de herramientas.

**Tokens de salida (*output tokens*).** Lo que el modelo genera. Suelen costar más que los de entrada.

**Tool (MCP).** Función que el modelo puede decidir invocar, con nombre, descripción y esquema de parámetros.

**Tool use (*function calling*).** Capacidad del modelo de pedir que la aplicación ejecute una función y le devuelva el resultado. El modelo no ejecuta nada por sí mismo.

**Ventana de contexto (*context window*).** Cantidad máxima de tokens que el modelo puede considerar en una llamada, sumando entrada y salida.
