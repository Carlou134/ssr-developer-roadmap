# Cómo Elegir un Estilo Arquitectónico

## No es una moda, es una decisión de trade-offs

No existe un estilo "mejor" en abstracto — cada uno de los estilos vistos en este punto (monolito, monolito modular, SOA, orientado a eventos, microservicios) resuelve bien ciertos problemas y paga un costo real por eso. Elegir uno porque "así lo hace Netflix" sin tener los problemas de escala de Netflix es el error de arquitectura más común y más caro de revertir.

## Preguntas que realmente deberían decidir esto

1. **¿Cuáles son mis NFR reales?** (visto en el [primer archivo de este punto](./01-requisitos-no-funcionales-y-restricciones.md)) — ¿de verdad necesito escalar partes del sistema de forma independiente, o es una necesidad hipotética?
2. **¿Cuál es el tamaño y madurez operativa de mi equipo?** Un equipo de 3 personas gestionando 12 microservicios va a pasar más tiempo operando infraestructura que construyendo producto.
3. **¿Qué tan bien entendido está el dominio del negocio?** Si los límites del negocio todavía están cambiando (proyecto nuevo, producto en validación), un monolito modular permite mover esos límites moviendo código dentro del mismo proyecto — dividir en servicios demasiado pronto congela límites que probablemente están mal, y corregirlos después es mucho más caro.
4. **¿Cuál es el presupuesto y su naturaleza (Capex vs. Opex)?**

## Capex vs. Opex, aplicado a decisiones de arquitectura

- **Capex (Capital Expenditure):** inversión inicial fuerte, por adelantado — ej. comprar servidores físicos propios.
- **Opex (Operational Expenditure):** costo recurrente, que escala con el uso — el modelo típico de la nube (pagas por lo que usas, mes a mes).

La mayoría de arquitecturas modernas en la nube son intensivas en Opex: cada microservicio adicional, cada base de datos separada, cada mensaje que cruza un bus de eventos, tiene un costo recurrente y medible. Esto no es un detalle financiero aislado del trabajo técnico — es una entrada directa a la decisión de arquitectura: una arquitectura de microservicios mal dimensionada puede generar una factura de nube significativamente mayor que un monolito bien optimizado, sin que el sistema entregue más valor de negocio por eso.

## Build vs. Buy

Antes de construir un componente de infraestructura propio (tu propio bus de mensajes, tu propio sistema de autenticación, tu propio motor de búsqueda), vale la pena preguntar si existe un servicio administrado que resuelva el mismo problema (Kafka administrado, Auth0/Azure AD B2C, Elasticsearch administrado). Construirlo propio solo se justifica cuando es parte del valor diferencial real del producto — para todo lo demás ("commodity" técnico que no es tu ventaja competitiva), comprar/usar un servicio administrado casi siempre gana en costo total (desarrollo + mantenimiento + on-call).

## Regla práctica de cierre

Empieza con lo más simple que cumpla tus NFR reales — casi siempre eso es un monolito modular bien construido con los principios del punto 2 (Clean Architecture, límites claros entre módulos). Migra hacia SOA, eventos o microservicios cuando un NFR concreto y medido (no una sospecha) lo exija, y hazlo de forma gradual (Strangler Fig, visto en el archivo de [Monolitos](./03-monolitos-y-monolito-modular.md)), no con una reescritura completa desde cero.
