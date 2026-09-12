# Documentar Decisiones Arquitectónicas

## El problema que resuelve

Seis meses después de elegir "monolito modular" o "microservicios", alguien nuevo en el equipo (o tú mismo) se pregunta: "¿por qué se decidió esto así?". Sin un registro, la respuesta se pierde o se reconstruye de memoria, con el riesgo de repetir un error ya descartado por una buena razón que nadie recuerda.

## ADR (Architectural Decision Record)

Un **ADR** es un documento corto (una o dos páginas) que registra una decisión de arquitectura específica en el momento en que se toma. El formato popularizado por Michael Nygard es simple a propósito:

- **Título:** la decisión en una frase (ej. "Usar PostgreSQL en vez de MongoDB para el servicio de Pedidos").
- **Contexto:** qué problema o fuerzas llevaron a necesitar esta decisión.
- **Decisión:** qué se decidió, en términos concretos.
- **Consecuencias:** qué se gana y qué se sacrifica con esta decisión — incluyendo las desventajas, no solo las ventajas.

Un ADR no se borra ni se reescribe cuando la decisión cambia — se agrega un ADR nuevo que referencia al anterior como "reemplazado". Esto conserva el historial de *por qué* se pensó así en su momento, con el contexto que existía entonces (que puede ser distinto al contexto actual).

## `ARCHITECTURE.md`: el mapa vivo del sistema

Mientras los ADRs registran decisiones puntuales en el tiempo, un archivo `ARCHITECTURE.md` en la raíz del repositorio mantiene una **vista actual** del sistema: qué servicios/módulos existen, cómo se comunican, dónde están los límites de dominio, y quizás un diagrama C4 de nivel Context o Containers (visto en el archivo de [Microservicios](./06-microservicios-ventajas-y-costos-ocultos.md)). A diferencia de un ADR, este documento sí se actualiza constantemente para reflejar el estado real del sistema — es fácil que quede desactualizado si no se trata como parte del trabajo normal (igual que la documentación de API del punto 3, que solo sirve si se mantiene viva).

## Fitness Functions: validar la arquitectura con pruebas automáticas

Un concepto del libro *"Building Evolutionary Architectures"* (Neal Ford, Rebecca Parsons, Patrick Kua): una **Fitness Function** es una prueba automatizada que valida que una característica arquitectónica se sigue cumpliendo, de la misma forma que un test unitario valida que una función se sigue comportando bien. Ejemplos concretos:

- Un test que falla en CI si el proyecto `Domain` termina referenciando al proyecto `Infrastructure` (rompiendo la Regla de Dependencia de Clean Architecture, vista en el punto 2) — herramientas como `NetArchTest` en .NET permiten escribir esto como un test real.
- Un test que mide el tiempo de respuesta de un endpoint crítico y falla el pipeline si supera un umbral de NFR definido.
- Un escaneo de seguridad automatizado (dependencias vulnerables) que bloquea el merge si aparece una vulnerabilidad crítica nueva.

La idea de fondo: las reglas de arquitectura que solo viven "en la cabeza del equipo" o en un documento que nadie relee se erosionan con el tiempo. Convertir la regla en un test automatizado la hace tan difícil de romper por accidente como cualquier otro test que ya corre en el pipeline de CI/CD.

## Una expectativa realista: esto se abandona seguido

Vale la pena ser honesto sobre esto en vez de venderlo como una práctica universal: ADRs, `ARCHITECTURE.md` y diagramas C4 son recomendaciones que casi todo el mundo repite, pero que en la mayoría de equipos reales se abandonan apenas hay presión de entregar — se escriben los primeros 2 o 3 ADRs con entusiasmo, y después se dejan de lado. Se sostienen en el tiempo sobre todo en tres contextos: empresas con compliance/auditoría real (bancos, seguros), organizaciones grandes con un rol de Arquitecto dedicado que las hace cumplir, o proyectos open source maduros. Fuera de eso, lo más común es tribal knowledge en la cabeza de 2-3 personas senior, y un diagrama C4 dibujado una vez para una presentación que queda desactualizado al poco tiempo. Las Fitness Functions escapan un poco a esto porque, al ser código que corre en CI, no dependen de que alguien se acuerde de mantenerlas — fallan solas si algo se rompe.
