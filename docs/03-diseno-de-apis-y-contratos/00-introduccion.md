# Fase 3: Diseño de APIs y Contratos Técnicos

## ¿Por qué importa?

Una API es un contrato con quien la consume — un frontend, una app móvil, u otro servicio. Ese contrato tiene que ser predecible: mismos verbos para las mismas operaciones, mismos códigos de estado para los mismos tipos de resultado, misma estructura de error siempre. Sin esa consistencia, cada consumidor tiene que "adivinar" cómo se comporta la API, leyendo el código en vez de la documentación.

## Contenido

1. [Estándares RESTful](./01-estandares-restful.md) — verbos HTTP, nombres de recursos, códigos de estado, estructura estándar de errores, paginación.
2. [GraphQL](./02-graphql.md) — qué problema resuelve, conceptos clave, y cuándo conviene sobre REST.
3. [Documentación Técnica](./03-documentacion-tecnica.md) — Swagger/OpenAPI en la práctica, DTOs como contrato, y su relación con los ADRs de punto 1.
