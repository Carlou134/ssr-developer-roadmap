# Requisitos No Funcionales y Restricciones

## Requisitos Funcionales (FR) vs. No Funcionales (NFR)

- **Funcionales (FR):** qué hace el sistema. "El usuario puede crear un pedido", "el sistema envía un correo de confirmación". Son los que normalmente aparecen como historias de usuario.
- **No funcionales (NFR):** cómo lo hace, en términos de calidad. Rendimiento, disponibilidad, escalabilidad, seguridad, mantenibilidad, usabilidad. "El sistema debe responder en menos de 300ms" o "debe soportar 10,000 usuarios concurrentes" son NFR.

La razón por la que este archivo va primero en este punto: **los NFR son los que realmente deciden el estilo de arquitectura, no los FR.** Un sistema con las mismas funciones exactas ("gestionar pedidos") puede ser un monolito perfectamente razonable para una tienda pequeña, o requerir microservicios y arquitectura orientada a eventos para un marketplace con millones de transacciones diarias — la diferencia no está en qué hace el sistema, sino en sus NFR de escala y disponibilidad.

## Riesgos vs. Restricciones

Dos conceptos que se confunden fácilmente:

- **Restricción (constraint):** un hecho dado, no negociable, que condiciona el diseño desde el inicio. Ejemplos: "el presupuesto de infraestructura es de $X al mes", "debe desplegarse en AWS porque ya es el proveedor de la empresa", "el equipo tiene 3 personas y ninguna conoce Kubernetes". Las restricciones no se gestionan, se aceptan y se diseña alrededor de ellas.
- **Riesgo:** un evento incierto que *podría* pasar y afectar el proyecto si ocurre. Ejemplo: "el proveedor de pagos externo podría tener downtime en horas pico". Los riesgos sí se gestionan — se puede mitigar (agregar un circuit breaker), transferir (un SLA con penalización), o aceptar conscientemente.

Un error común de Junior a Semi-Senior es tratar una restricción como si fuera negociable (proponer una solución técnicamente ideal que ignora el presupuesto o el tamaño real del equipo), o al revés, tratar un riesgo como si fuera un hecho fijo e inevitable sin evaluar si vale la pena mitigarlo.

## Por qué esto importa antes de elegir un estilo arquitectónico

Antes de preguntarte "¿monolito o microservicios?", la pregunta que realmente responde eso es: ¿cuáles son mis NFR reales (no los que "suenan bien" tener) y cuáles son mis restricciones reales de equipo y presupuesto? Elegir un estilo arquitectónico sin haber hecho explícitos estos dos puntos es la causa más común de sobre-ingeniería — vas a ver esto de nuevo en el archivo [Cómo Elegir un Estilo Arquitectónico](./07-como-elegir-un-estilo-arquitectonico.md).
