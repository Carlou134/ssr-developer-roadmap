# SOA y Contratos de Servicio

## La idea central de SOA

**Arquitectura Orientada a Servicios (SOA)** organiza un sistema como un conjunto de servicios que se comunican mediante **contratos bien definidos** — cada servicio expone una interfaz explícita que describe qué operaciones ofrece, qué datos espera y qué devuelve, sin que quien lo consume necesite saber cómo está implementado por dentro.

Esta idea de "contrato explícito entre servicios" no desapareció — es la misma idea de fondo detrás de un contrato OpenAPI hoy. Lo que cambió con el tiempo fue la tecnología usada para expresar ese contrato, no el concepto.

## El stack clásico de SOA (y por qué es importante conocerlo, aunque ya no se use)

SOA, en su forma clásica de los 2000s, se apoyaba en un stack basado en XML:

- **SOAP:** el protocolo de mensajería, con mensajes XML pesados y estructurados.
- **WSDL (Web Services Description Language):** el documento que describe el contrato del servicio — qué operaciones existen, qué parámetros recibe cada una.
- **XSD (XML Schema Definition):** define la estructura y tipos de datos válidos dentro de los mensajes XML.

**Aclaración importante:** OpenAPI/Swagger (lo que usas hoy, visto en el [punto 3](../03-diseno-de-apis-y-contratos/03-documentacion-tecnica.md)) **no es la versión moderna de WSDL en el sentido de que ambos convivieron en la misma época** — OpenAPI pertenece a la generación posterior de APIs REST/JSON, no al stack SOA/SOAP. Si ves material que los presenta como "equivalentes históricos" en la misma línea de tiempo, es una simplificación que mezcla dos generaciones distintas de tecnología. Lo que sí es correcto decir es que **resuelven el mismo problema de fondo** (documentar un contrato de forma que otros puedan integrarse sin adivinar) con herramientas de época distintas.

## Por qué se migró de SOAP a REST/JSON

- **Peso:** XML es más verboso que JSON — más bytes por el cable, más costo de parseo.
- **Complejidad de tooling:** generar y consumir un cliente SOAP a partir de un WSDL requería herramientas específicas por lenguaje; REST + JSON se consume con un simple `fetch`/`HttpClient` sin generación de código obligatoria.
- **Legibilidad humana:** un JSON se lee a simple vista; un mensaje SOAP con namespaces XML es mucho más difícil de inspeccionar a mano durante debugging.
- **Ecosistema web:** REST encaja naturalmente con el protocolo HTTP que ya usa el navegador, sin necesitar una capa adicional de envoltorio como hace SOAP.

## Dónde sigue viva la idea de SOA hoy

Aunque SOAP/WSDL casi no se usa en proyectos nuevos, el **concepto** de SOA (servicios con contratos explícitos, reutilizables entre distintos consumidores) sigue totalmente vigente — solo que hoy se implementa con REST + OpenAPI, gRPC + Protocol Buffers, o GraphQL (que ya viste en el punto 3). Sistemas gubernamentales o bancarios de integración entre organizaciones (facturación electrónica, interoperabilidad entre bancos) todavía usan SOAP/XML en muchos países, porque migrar un contrato ya establecido entre múltiples organizaciones externas es mucho más costoso que hacerlo dentro de una sola empresa.

## Caso real: bancos y el Core Bancario

El escenario más probable de encontrarte esto en la práctica (y en entrevistas de empresas financieras): un banco tiene un **Core Bancario** legado (plataformas como Temenos T24, FIS, Oracle FLEXCUBE) que expone sus operaciones vía SOAP/WSDL, y te piden crear un microservicio moderno (Spring Boot o .NET) que **consuma** ese servicio viejo — no que crees uno SOAP nuevo desde cero, eso ya casi no se hace.

**La causa real de por qué sigue ahí no es "se eligió por seguridad" — es inercia y riesgo de migración.** Es fácil malinterpretar esto: WS-Security (seguridad a nivel de mensaje, no solo de transporte) y WS-AtomicTransaction (transacciones distribuidas reales, algo que REST no replica de forma nativa) son capacidades técnicas reales y legítimas de SOAP — pero eso no significa que el banco "decidió" seguir usando SOAP por eso. La razón de fondo es que reemplazar un Core Bancario es carísimo y riesgoso (es el sistema más crítico de la organización), así que ese contrato de hace 15-20 años se queda ahí, y las nuevas piezas del sistema se construyen alrededor consumiéndolo, en vez de reemplazarlo.

**Tooling real para consumir un WSDL hoy (esto sí cambia según el ecosistema):**

- **.NET:** la herramienta moderna es `dotnet-svcutil` (reemplaza al viejo `svcutil.exe` de .NET Framework), que Visual Studio envuelve con la opción "Connected Services → WCF Web Service Reference". Genera las clases cliente a partir del WSDL automáticamente. **Matiz importante:** esto sirve para *consumir* un servicio SOAP existente. *Hospedar* (crear) uno nuevo en .NET moderno (Core/5+/8) no viene de fábrica — WCF del lado servidor se quedó en .NET Framework — hace falta el proyecto **CoreWCF** si alguna vez hay que exponer uno nuevo.
- **Java/Spring:** el equivalente es `wsimport` (parte de JAX-WS, ya no viene incluido desde el JDK 11 en adelante, hay que agregarlo como dependencia aparte) o, más común en proyectos Spring Boot reales, el plugin `cxf-codegen` de **Apache CXF** en Maven/Gradle, que genera las clases del cliente en tiempo de build a partir del WSDL.

En ambos casos, la idea clave para no asustarse en una entrevista: nadie escribe el XML del contrato a mano — se importa el WSDL, se genera el código, y se trabaja con clases normales del lenguaje.
