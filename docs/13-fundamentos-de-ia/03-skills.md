# Skills

## En una frase

Una skill es **una carpeta con instrucciones (`SKILL.md`), scripts y recursos que le enseña a un agente a hacer una tarea concreta**, y que el agente carga **solo cuando la necesita**, así no gasta contexto en lo que no usa.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es la ventana de contexto: [Tokens y ventana de contexto](01-tokens-y-ventana-de-contexto.md).
* Qué es tool use y MCP: [MCP (Model Context Protocol)](02-mcp-model-context-protocol.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Skill (*Agent Skill*):** paquete de conocimiento procedimental que un agente puede cargar bajo demanda.
* **`SKILL.md`:** archivo principal de una skill: metadatos en YAML más instrucciones en Markdown.
* **Frontmatter:** bloque YAML al inicio de un archivo Markdown, entre líneas `---`.
* **Carga progresiva (*progressive disclosure*):** cargar la información en niveles, de lo mínimo a lo completo, solo cuando hace falta.
* **Prompt de sistema (*system prompt*):** instrucciones que acompañan siempre a la conversación.
* **Conocimiento procedimental:** saber **cómo** hacer algo (pasos, convenciones, criterios), no solo **qué** es.

-----

## El problema

Tu equipo usa un agente de código para crear las notas de versión. Cada vez alguien pega en el chat las mismas reglas: formato de *conventional commits*, secciones obligatorias, cómo nombrar los *breaking changes*, el script que lista los PRs. Para no repetirlas, las ponen en el prompt de sistema junto con las reglas de testing, de migraciones, de code review…

```text
Prompt de sistema (siempre cargado):
  - reglas de release notes        (2.500 tokens)
  - guía de migraciones EF Core     (3.000 tokens)
  - checklist de code review        (1.800 tokens)
  - convenciones de testing         (2.200 tokens)
  - ... 10 guías más
```

Las consecuencias:

* cada conversación paga miles de tokens de guías que no va a usar;
* las instrucciones relevantes se diluyen entre las irrelevantes y el agente las sigue peor;
* las reglas viven en un texto gigante que nadie mantiene ni versiona;
* el conocimiento no se puede reutilizar en otro agente o en otro proyecto.

-----

## Cómo funciona

### 1. Qué hay dentro de una skill

Una skill es una carpeta:

```text
release-notes/
├── SKILL.md            ← obligatorio: metadatos + instrucciones
├── template.md         ← opcional: recursos de referencia
└── scripts/
    └── list-prs.sh     ← opcional: código que el agente puede ejecutar
```

`SKILL.md` empieza con un frontmatter con dos campos obligatorios:

```markdown
---
name: release-notes
description: Genera notas de versión a partir de los PRs mergeados. Úsala cuando el usuario pida release notes, changelog o resumen de una versión.
---

# Release notes

1. Ejecuta `scripts/list-prs.sh <tag-anterior>` para obtener los PRs.
2. Agrupa por tipo de conventional commit: feat, fix, perf.
3. Sigue el formato de `template.md`.
...
```

### 2. Carga progresiva: la idea central

Las skills no se cargan enteras. Hay tres niveles:

| Nivel | Qué se carga | Cuándo | Costo aproximado |
| --- | --- | --- | --- |
| 1. Metadatos | `name` + `description` | Siempre, al iniciar | Decenas de tokens por skill |
| 2. Instrucciones | El cuerpo de `SKILL.md` | Cuando el agente decide que la skill es relevante | Lo que mida el archivo |
| 3. Recursos | Otros archivos y scripts | Solo si las instrucciones los necesitan | Variable; un script ejecutado solo aporta su salida |

Por eso puedes tener cien skills instaladas sin llenar el contexto: el agente ve un índice corto y abre solo lo que necesita, como un manual con índice en lugar de leerlo entero cada mañana.

> **Ojo:** el nivel 1 es lo **único** que el agente ve para decidir. Si la `description` no dice **cuándo** usar la skill, el agente nunca la va a abrir.

Si lo usas mal:

```markdown
---
name: helper
description: Ayuda con cosas del proyecto.
---
```

```text
Usuario: "Arma las release notes de la v2.3"
Agente:  (no relaciona la petición con "helper"; improvisa un formato propio)
```

No hay error visible: la skill simplemente no se activa. Es un fallo silencioso.

### 3. Los scripts: código determinista

Para tareas que deben salir siempre igual (contar, validar, transformar archivos), la skill incluye un script y el agente lo ejecuta en lugar de "razonar" el resultado. Es más barato en tokens y más confiable. Esto requiere que el agente tenga un entorno donde ejecutar código.

### 4. Skills frente a las otras piezas

| Pieza | Qué aporta | Cuándo se carga | Analogía |
| --- | --- | --- | --- |
| **Prompt de sistema / `CLAUDE.md`** | Reglas generales del proyecto | Siempre | El reglamento de la obra |
| **Skill** | Cómo hacer una tarea concreta | Bajo demanda | El manual de un procedimiento |
| **MCP** | Acceso a sistemas externos (tools, datos) | Definiciones al conectar; datos al invocar | Las herramientas y llaves de acceso |
| **Subagente** | Otra instancia con contexto propio | Cuando se delega una tarea | Un contratista independiente |

La frase para recordar: **MCP le da al agente manos; una skill le enseña el oficio.** Se combinan: una skill puede indicar qué herramientas MCP usar y en qué orden.

-----

## Ejemplo completo

Resolvemos el problema: sacamos las reglas de release notes del prompt de sistema y las convertimos en una skill.

`.claude/skills/release-notes/SKILL.md`:

```markdown
---
name: release-notes
description: Genera notas de versión en español a partir de los PRs mergeados desde el último tag. Úsala cuando pidan release notes, changelog, notas de versión o un resumen de lo que entró en una versión.
---

# Release notes

## Pasos

1. Pide el tag anterior si el usuario no lo indicó.
2. Ejecuta `scripts/list-prs.sh <tag>`. Devuelve un PR por línea: `tipo|titulo|numero`.
3. Agrupa por tipo: `feat` → Novedades, `fix` → Correcciones, `perf` → Rendimiento.
   Ignora `chore`, `docs` y `test`.
4. Si algún título contiene `!` o `BREAKING CHANGE`, crea una sección **Cambios incompatibles** al inicio.
5. Usa exactamente el formato de `template.md`.

## Reglas

- Una línea por cambio, en lenguaje de negocio, sin jerga interna.
- Siempre enlaza el número de PR.
```

`.claude/skills/release-notes/scripts/list-prs.sh`:

```bash
#!/usr/bin/env bash
# Lista los commits de merge desde el tag indicado como "tipo|titulo|numero"
git log "$1"..HEAD --merges --pretty=format:'%s' |
  sed -E 's/^Merge pull request #([0-9]+).*$/\1/' |
  while read -r pr; do
    gh pr view "$pr" --json title,number --jq '"\(.title|split(":")[0])|\(.title)|\(.number)"'
  done
```

Resultado: el prompt de sistema pierde 2.500 tokens. Cuando alguien escribe "armame el changelog de la v2.3", el agente ve en el índice la `description` de `release-notes`, carga la skill, ejecuta el script y entrega el documento con el formato del equipo. En las demás conversaciones, la skill solo ocupa su línea en el índice.

Qué observar:

* la `description` incluye **qué hace** y **cuándo usarla**, con los sinónimos que usaría una persona;
* lo determinista (listar PRs) va en un script; el criterio (redactar en lenguaje de negocio) queda para el modelo;
* el formato vive en `template.md` y solo se lee en el paso 5;
* la skill está en el repositorio: se versiona, se revisa en PR y la usa todo el equipo.

-----

## Errores comunes

**1. Description que no dice cuándo usarla.**
Qué pasa: la skill nunca se activa y no hay ningún error.
Por qué: el agente decide solo con `name` y `description`.
Arreglo: escribe qué hace y en qué situaciones usarla, con las palabras que usaría el usuario.

**2. Meter todo en `SKILL.md`.**
Qué pasa: al activarse, la skill carga miles de tokens innecesarios.
Por qué: se pierde la ventaja del nivel 3.
Arreglo: deja en `SKILL.md` los pasos y mueve referencias largas a archivos aparte, enlazados desde las instrucciones.

**3. Confundir skill con MCP.**
Qué pasa: se crea un servidor MCP para "enseñar" un formato, o una skill para acceder a una API sin credenciales.
Por qué: MCP da acceso y capacidades; la skill da procedimiento.
Arreglo: si necesitas **conectarte** a algo, MCP. Si necesitas **cómo hacer** algo, skill.

**4. Instalar skills de terceros sin leerlas.**
Qué pasa: una skill ejecuta scripts que no conoces o contiene instrucciones maliciosas.
Por qué: una skill es código y prompt que el agente va a obedecer.
Arreglo: revísala como revisarías una dependencia: solo de fuentes confiables y leyendo los scripts.

-----

## Evolución y versiones (opcional)

* **Octubre 2025:** Anthropic lanza Agent Skills en Claude (apps, Claude Code y API).
* **Diciembre 2025:** el formato se publica como estándar abierto y otras herramientas de agentes empiezan a soportarlo.

-----

## Cuándo sí y cuándo no

**Úsalo cuando:** tienes un procedimiento que se repite, con pasos, formato o criterios propios del equipo (release notes, migraciones, generar reportes, crear documentos con la marca de la empresa), y no quieres pagarlo en tokens en cada conversación.

**No lo uses cuando:** la regla aplica **siempre** (va en el prompt de sistema o `CLAUDE.md`), necesitas acceso a un sistema externo (va en MCP) o es algo de una sola vez (basta con pedirlo en el chat).

-----

## Resumen en 5 líneas

1. Una skill es una carpeta con `SKILL.md` (frontmatter `name` + `description` e instrucciones), más scripts y recursos opcionales.
2. Se carga en tres niveles: metadatos siempre, instrucciones al activarse, recursos bajo demanda.
3. La `description` decide si se activa: debe decir qué hace y cuándo usarla.
4. MCP le da acceso a sistemas; una skill le enseña un procedimiento. Se complementan.
5. Usa skills para procedimientos repetibles; las reglas permanentes van en el prompt de sistema.

-----

## Para profundizar (opcional)

<details>
<summary>¿Por qué no basta con una buena carpeta de prompts?</summary>

Una carpeta de prompts depende de que la persona elija y pegue el correcto. Con skills, **el agente** decide cuándo cargar cada una a partir del índice, y puede ejecutar los scripts asociados. Además, el formato es portable entre herramientas que soportan el estándar.

</details>

<details>
<summary>¿Qué relación tienen las skills con el "context engineering"?</summary>

*Context engineering* es la disciplina de decidir qué entra al contexto del modelo en cada momento. Las skills son una aplicación directa: en lugar de cargar todo de antemano, se carga el mínimo y se amplía bajo demanda. Lo mismo hacen RAG con documentos, la carga diferida de herramientas MCP y los subagentes que trabajan en un contexto aislado y devuelven solo el resultado.

</details>

-----

## En entrevista (opcional)

### Respuesta corta (junior)

Una skill es una carpeta con un archivo `SKILL.md` que contiene instrucciones para que un agente haga una tarea específica, más scripts o archivos de apoyo. El agente la carga solo cuando la tarea lo necesita.

### Respuesta ampliada (semi-senior)

Las skills empaquetan conocimiento procedimental: pasos, formatos, criterios y scripts de una tarea. Su clave es la carga progresiva: el agente siempre ve solo el `name` y la `description` de cada skill, carga el cuerpo cuando la tarea coincide y lee los archivos adicionales solo si hace falta. Así puedes tener muchas sin pagar su costo en tokens ni diluir el contexto. Se diferencian de MCP en que MCP da **acceso** a sistemas externos, mientras que la skill enseña **cómo** hacer algo; y del prompt de sistema en que este se carga siempre. Los riesgos: una `description` mal escrita hace que nunca se active, y una skill de terceros es código y prompt que el agente ejecuta, así que se audita como una dependencia.

### Preguntas frecuentes de seguimiento

**1. Si tengo 50 skills, ¿consumo el contexto de 50 skills?**
No. Solo consumes sus metadatos (nombre y descripción); el cuerpo se carga cuando una se activa.

**2. ¿Skill o MCP para integrar con Jira?**
MCP para conectarse a Jira (crear y leer issues con credenciales). Una skill si además quieres enseñar **cómo** escribe tu equipo los issues: plantilla, etiquetas, criterios de aceptación. Lo ideal suele ser ambas.

**3. ¿Por qué incluir scripts y no dejar que el modelo lo haga?**
Porque el código es determinista, más barato en tokens y no "alucina" un cálculo. El modelo queda para lo que requiere criterio.

-----

## Práctica

**Ejercicio 1.** Clasifica cada necesidad como **prompt de sistema / `CLAUDE.md`**, **skill** o **MCP**:
(a) "Todas las respuestas en español". (b) "Consultar los logs de producción en Seq". (c) "Cuando creemos una migración de EF Core, seguir estos 6 pasos y correr este script de validación".

<details>
<summary>Solución</summary>

(a) **Prompt de sistema / `CLAUDE.md`**: aplica siempre y es corto. (b) **MCP**: requiere acceso a un sistema externo con credenciales. (c) **Skill**: es un procedimiento con pasos y un script, que solo se necesita al crear migraciones.

</details>

**Ejercicio 2.** ¿Qué tiene de malo esta skill?

```markdown
---
name: utils
description: Utilidades varias del equipo.
---

(4.000 líneas con la guía de estilo, el proceso de deploy, las plantillas de PR
y la documentación completa de la API interna)
```

<details>
<summary>Solución</summary>

La descripción no dice qué hace ni cuándo usarla, así que el agente casi nunca la va a activar, y cuando lo haga, va a cargar las 4.000 líneas de una vez. Además mezcla tareas sin relación. Arreglo: dividirla en skills con un propósito cada una (`deploy`, `pull-request`, etc.), cada `description` con qué y cuándo, y mover la documentación de la API a archivos de referencia que se lean solo si hacen falta.

</details>

-----

## Siguiente lección

Volver al [inicio de la fase](README.md).
