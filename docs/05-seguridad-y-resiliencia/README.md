# Fase 5: Seguridad y resiliencia (DevSecOps básico)

En esta carpeta aprendes lo que un Semi-Senior tiene que saber para **no introducir vulnerabilidades y para que el sistema siga en pie cuando algo falla**: los principios de seguridad, el OWASP Top 10 actual, el control de acceso (BOLA y RBAC), la inyección y el XSS, la autenticación y los datos sensibles, la gestión de secretos, y los patrones de resiliencia: timeouts, retries, circuit breaker y rate limiting.

-----

## Por qué importa

La seguridad no es una funcionalidad que se agrega al final ni un tema exclusivo de un equipo especializado. La mayoría de las vulnerabilidades reales vienen de decisiones cotidianas de desarrollo: un endpoint que no verifica de quién es el recurso, una consulta concatenada, un secreto en el repositorio, un `catch` que deja pasar todo. Y la mayoría de las caídas no vienen de tu código, sino de una dependencia lenta que nadie limitó con un timeout.

Un Junior hace que funcione. Un Semi-Senior hace que funcione **cuando alguien intenta romperlo** y **cuando lo que lo rodea falla**.

-----

## Antes de empezar

Conviene que ya tengas:

* Una API con ASP.NET Core, controllers y Entity Framework Core.
* Códigos de estado y Problem Details: [Estándares RESTful](../03-diseno-de-apis-y-contratos/01-estandares-restful.md).
* Inyección de dependencias y el patrón Options: [Inyección de dependencias](../02-calidad-de-codigo-y-arquitectura/06-inyeccion-de-dependencias.md).
* Logging estructurado: [Interpretación de logs](../01-autonomia-resolucion-problemas/03-interpretacion-de-logs.md).

Si en algún momento aparece una palabra que no conoces, búscala en el [Glosario](Glosario.md).

-----

## Orden de lectura

| Lección | Qué aprendes | Qué necesitas saber antes |
| --- | --- | --- |
| [1. Fundamentos de seguridad web](01-fundamentos-de-seguridad.md) | Tríada CIA, defensa en profundidad, mínimo privilegio, modelado de amenazas con STRIDE y DevSecOps | APIs REST |
| [2. OWASP Top 10 (2025)](02-owasp-top-10.md) | Las diez categorías vigentes, cómo cambiaron desde 2017, SSRF, cadena de suministro, fallar cerrado y el API Security Top 10 | Fundamentos |
| [3. Control de acceso: BOLA y RBAC](03-control-de-acceso-bola-y-rbac.md) | Autenticación frente a autorización, BOLA, asignación masiva, roles, políticas, denegar por defecto y autorización basada en recursos | OWASP Top 10 |
| [4. Inyección, XSS y sanitización](04-inyeccion-xss-y-sanitizacion.md) | SQLi y consultas parametrizadas en EF Core, command injection, XSS, codificación de salida, sanitización y CSP | Fundamentos |
| [5. Autenticación y datos sensibles](05-autenticacion-y-datos-sensibles.md) | Hash de contraseñas, fuerza bruta, cookies seguras, JWT, CSRF, HTTPS y cifrado en reposo | Control de acceso, XSS |
| [6. Gestión de secretos](06-gestion-de-secretos.md) | Configuración frente a secretos, User Secrets, variables de entorno, Key Vault con identidad administrada y qué hacer ante una filtración | Patrón Options |
| [7. Resiliencia con Polly](07-resiliencia-con-polly.md) | Timeouts, retries con backoff y jitter, idempotencia, circuit breaker, fallback y `AddStandardResilienceHandler` | Idempotencia, DI |
| [8. Rate limiting](08-rate-limiting.md) | Algoritmos, el middleware de ASP.NET Core, particiones, proxies, `429` con `Retry-After` y varias instancias | Autenticación, resiliencia |

-----

## El mapa completo en una mirada

```text
PRINCIPIOS
  CIA                → confidencialidad · integridad · disponibilidad
  Regla de oro       → todo lo que viene de afuera (incluida la URL) es hostil
  Capas              → defensa en profundidad · mínimo privilegio · denegar por defecto · fallar cerrado
  Proceso            → STRIDE al diseñar · secretos, SAST y dependencias escaneados en cada PR

QUIÉN ERES Y QUÉ PUEDES HACER
  Autenticación      → hash lento con salt · bloqueo por intentos · MFA · cookies HttpOnly/Secure/SameSite
  Autorización       → dueño del recurso desde los claims (BOLA) · DTO de entrada (asignación masiva)
                        · políticas por rol (RBAC) · fallback policy · verificado SIEMPRE en el servidor

DATOS QUE SE VUELVEN CÓDIGO
  SQL                → LINQ / FromSql con interpolación · nunca FromSqlRaw con $"..."
  Navegador (XSS)    → codificar la salida (React y Razor lo hacen) · sanitizar solo si hace falta HTML · CSP

LO QUE NO VA EN EL CÓDIGO
  Secretos           → User Secrets (dev) · Key Vault + identidad administrada (prod) · si se filtra: ROTAR

CUANDO ALGO FALLA (resiliencia)
  Dependencias       → timeout por intento y total · retry transitorio e idempotente con jitter · circuit breaker
  Abuso              → rate limiting particionado · 429 + Retry-After · contadores por instancia
```

-----

## De dónde viene este material

Esta fase parte de mis apuntes de seguridad web (introducción a la seguridad y el OWASP Top 10 de 2017), actualizados a la edición **2025** del Top 10 y ampliados con lo que pide el roadmap: BOLA, RBAC, sanitización, gestión de secretos y patrones de resiliencia. Los ejemplos de SQLi, XSS, XXE, autenticación rota y logs de los apuntes originales están integrados en las lecciones 2, 4 y 5.

-----

## Cómo está armada cada lección

Todas las lecciones tienen la misma estructura, así sabes dónde buscar cada cosa:

1. **En una frase:** qué es, sin jerga.
2. **Antes de empezar:** qué tienes que saber y las palabras nuevas.
3. **El problema:** una situación concreta que muestra por qué existe la defensa.
4. **Cómo funciona:** el paso a paso, con ejemplos cortos y explicados.
5. **Ejemplo completo:** todo junto, aplicado al mismo problema.
6. **Errores comunes:** lo que suele salir mal, por qué pasa y cómo se arregla.
7. **Evolución y versiones:** qué cambió con el tiempo, para reconocer material desactualizado.
8. **Cuándo sí y cuándo no:** para decidir si es la herramienta correcta.
9. **Resumen en 5 líneas:** para repasar.
10. **Para profundizar:** detalles avanzados, plegados. Puedes omitirlos en la primera lectura.
11. **En entrevista:** una respuesta corta (junior), una ampliada (semi-senior) y preguntas de seguimiento.
12. **Práctica:** uno o dos ejercicios con la solución plegada. Intenta resolverlos antes de mirar.
13. **Siguiente lección:** hacia dónde continuar.

> **Sobre la práctica:** los ejemplos de ataques de esta fase son para entender cómo funcionan las defensas. Pruébalos solo en tus proyectos locales o en entornos hechos para eso (OWASP Juice Shop, PortSwigger Web Security Academy, DVWA). Atacar un sistema sin autorización es ilegal.

-----

## Después de esta carpeta

El siguiente punto del [roadmap](../../README.md) es bases de datos y rendimiento.
