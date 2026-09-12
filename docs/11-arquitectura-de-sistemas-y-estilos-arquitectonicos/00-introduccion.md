# Arquitectura de Sistemas y Estilos Arquitectónicos

## ¿En qué se diferencia esto del punto 2?

El [punto 2](../02-calidad-de-codigo-y-arquitectura/) (Calidad de Código y Arquitectura) responde preguntas dentro de **un solo servicio**: cómo organizar las clases (SOLID), cómo separar capas dentro de un proyecto (Clean Architecture, Hexagonal), cómo separar lecturas de escrituras (CQRS). Es arquitectura **de aplicación**.

Este punto responde una pregunta distinta, un nivel más arriba: **¿cuántos procesos o servicios tiene mi sistema, y cómo se comunican entre ellos?** Es arquitectura **de sistemas**. Un mismo proyecto puede tener una Clean Architecture impecable por dentro y, aun así, estar mal decidido a nivel de sistema (por ejemplo, partido en 15 microservicios cuando un monolito modular habría sido más simple y más barato de operar).

Para un Junior esta distinción no suele importar — recibe la decisión ya tomada. Para nivel Semi-Senior empieza a importar, porque empiezas a opinar en conversaciones del tipo "¿este módulo nuevo lo metemos en el monolito o lo sacamos a un servicio aparte?".

## Contenido

1. [Requisitos No Funcionales y Restricciones](./01-requisitos-no-funcionales-y-restricciones.md) — FR vs. NFR, riesgos vs. restricciones, por qué los NFR son los que realmente empujan la decisión de arquitectura.
2. [Cliente-Servidor y Arquitectura en Capas](./02-cliente-servidor-y-arquitectura-en-capas.md) — el modelo base sobre el que se construye casi todo lo demás.
3. [Monolitos y Monolito Modular](./03-monolitos-y-monolito-modular.md) — ventajas y desventajas reales, sin mitos.
4. [SOA y Contratos de Servicio](./04-soa-y-contratos-de-servicio.md) — de WSDL/SOAP a REST, y por qué cambió.
5. [Arquitectura Orientada a Eventos](./05-arquitectura-orientada-a-eventos.md) — productores, consumidores, bus de mensajes, y sus retos reales.
6. [Microservicios: Ventajas y Costos Ocultos](./06-microservicios-ventajas-y-costos-ocultos.md) — lo que se gana y lo que casi nadie cuenta que se paga.
7. [Cómo Elegir un Estilo Arquitectónico](./07-como-elegir-un-estilo-arquitectonico.md) — un marco de decisión, no una moda.
8. [Documentar Decisiones Arquitectónicas](./08-documentar-decisiones-arquitectonicas.md) — ADRs, C4 Model, Fitness Functions, y qué tanto se sostiene esto en la práctica real.
9. [Hoja de Referencia: Cuándo Usar Cada Arquitectura](./09-cuando-usar-cada-arquitectura.md) — tabla resumen con un ejemplo claro por cada arquitectura vista, tanto del punto 2 como de este punto.
10. [Prompts Útiles](./10-prompts-utiles.md) — plantillas para evaluar trade-offs de arquitectura con ayuda de IA.
