# Prompts Útiles: Arquitectura de Sistemas y Estilos Arquitectónicos

## Evaluar monolito vs. microservicios para un caso concreto

> Estoy decidiendo la arquitectura de sistema para [describir el proyecto: qué hace, tamaño del equipo, escala esperada]. Ayúdame a evaluar monolito modular vs. microservicios identificando primero mis requisitos no funcionales reales (escala, disponibilidad, independencia de despliegue) y mis restricciones (tamaño de equipo, presupuesto, madurez operativa). Dame una recomendación justificada, no una lista genérica de pros y contras.

## Redactar un ADR

> Tomamos esta decisión de arquitectura: [describir la decisión y la alternativa descartada]. Ayúdame a redactar un ADR siguiendo el formato de Michael Nygard (Contexto, Decisión, Consecuencias), siendo honesto también con las desventajas de la opción elegida, no solo sus ventajas.

## Definir límites de dominio para arquitectura orientada a eventos

> Tengo estos servicios/dominios en mi sistema: [listar]. Ayúdame a identificar qué eventos debería publicar cada uno, evitando que el lenguaje de un dominio se filtre incorrectamente hacia otro (Bounded Context), y señala si algún evento propuesto en realidad esconde una necesidad de respuesta síncrona (y por tanto no debería ser un evento).

## Revisar si una decisión de arquitectura es sobre-ingeniería

> Estoy considerando [SOA / microservicios / arquitectura orientada a eventos] para este caso: [describir el contexto real]. Antes de decidir, cuestiona si esto es necesario ahora mismo o si es sobre-ingeniería — compáralo con la opción más simple (monolito modular) y dime qué NFR concreto justificaría de verdad la complejidad adicional.
