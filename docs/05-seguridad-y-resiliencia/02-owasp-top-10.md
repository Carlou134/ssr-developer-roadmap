# OWASP Top 10 (2025)

## En una frase

El **OWASP Top 10** es la lista de las diez categorías de riesgo más críticas en aplicaciones web, armada con datos reales de miles de aplicaciones; no es un checklist completo, sino **el mapa mínimo que todo desarrollador tiene que reconocer** en su propio código, y en su edición 2025 lo encabeza el control de acceso roto.

-----

## Antes de empezar

Conviene que ya sepas:

* La tríada CIA y la defensa en profundidad: [Fundamentos de seguridad web](01-fundamentos-de-seguridad.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **OWASP (*Open Worldwide Application Security Project*):** fundación sin fines de lucro que publica guías, herramientas y estándares de seguridad de aplicaciones.
* **CWE (*Common Weakness Enumeration*):** catálogo de tipos de debilidades de software, como "CWE-89: SQL Injection".
* **CVE (*Common Vulnerabilities and Exposures*):** identificador público de una vulnerabilidad concreta en un producto concreto, como `CVE-2021-44228` (Log4Shell).
* **Inyección:** cuando datos del usuario terminan interpretados como código por un intérprete (SQL, sistema operativo, navegador).
* **XSS (*Cross-Site Scripting*):** inyección de JavaScript que se ejecuta en el navegador de otro usuario.
* **SSRF (*Server-Side Request Forgery*):** lograr que el servidor haga peticiones a destinos elegidos por el atacante.
* **XXE (*XML External Entities*):** abuso de un parser XML para leer archivos internos o hacer peticiones.
* **Deserialización insegura:** convertir datos no confiables en objetos de forma que el atacante controle qué se crea o se ejecuta.
* **Cadena de suministro (*supply chain*):** todo lo que interviene en construir y distribuir tu software: dependencias, herramientas de build, pipelines, registros de paquetes.
* **Fail open (fallar abierto):** ante un error, el sistema permite el acceso en lugar de negarlo.

-----

## El problema

Un equipo estudió seguridad con un curso de 2017. Revisan su API buscando "los 10 de OWASP" y concluyen que está bien: no usan XML, así que XXE no aplica, y escapan el HTML.

Lo que no revisaron, porque en su lista no aparecía o aparecía con otro nombre:

* el endpoint `/api/preview?url=` descarga cualquier URL para generar una vista previa, incluida `http://169.254.169.254/` (los metadatos de la nube, con credenciales temporales): **SSRF**, que en 2017 no estaba en el Top 10;
* el pipeline de CI descarga un script de instalación con `curl | bash` desde un dominio que expiró: **cadena de suministro**;
* el middleware de autorización tiene un `try/catch` que, si falla la consulta de permisos, deja pasar la petición: **manejo incorrecto de condiciones excepcionales**, nueva en 2025.

Las consecuencias: una lista desactualizada da una falsa sensación de seguridad. El mapa cambia porque los ataques cambian.

-----

## Cómo funciona

### 1. Qué es (y qué no es) el Top 10

El Top 10 lo arma OWASP con datos de pruebas sobre cientos de miles de aplicaciones y con una encuesta a la comunidad de seguridad. Cada categoría agrupa varias debilidades (CWE) relacionadas.

| Es | No es |
| --- | --- |
| Un documento de concientización con los riesgos más comunes y graves | Un estándar de seguridad completo |
| Un mapa para priorizar qué aprender y qué revisar primero | Un checklist: "cumplir el Top 10" no hace segura una aplicación |
| Una referencia común en entrevistas, auditorías y herramientas | Una lista fija: se actualiza cada tres o cuatro años |

Para requisitos verificables, OWASP publica el **ASVS** (*Application Security Verification Standard*). Para APIs, existe el **OWASP API Security Top 10**, que se ve más abajo.

### 2. La lista de 2025

| # | Categoría | En una línea | Dónde se profundiza |
| --- | --- | --- | --- |
| **A01** | Broken Access Control | El usuario accede a datos o acciones que no le corresponden | [Control de acceso](03-control-de-acceso-bola-y-rbac.md) |
| **A02** | Security Misconfiguration | Configuraciones inseguras o por defecto | [Gestión de secretos](06-gestion-de-secretos.md) |
| **A03** | Software Supply Chain Failures | Dependencias, builds o pipelines comprometidos | Esta lección |
| **A04** | Cryptographic Failures | Datos sensibles sin cifrar o con criptografía débil | [Autenticación y datos sensibles](05-autenticacion-y-datos-sensibles.md) |
| **A05** | Injection | Datos del usuario interpretados como código (SQL, comandos, XSS) | [Inyección, XSS y sanitización](04-inyeccion-xss-y-sanitizacion.md) |
| **A06** | Insecure Design | Fallas de diseño que ningún código "bien escrito" corrige | Esta lección |
| **A07** | Authentication Failures | Suplantar a otro usuario | [Autenticación y datos sensibles](05-autenticacion-y-datos-sensibles.md) |
| **A08** | Software or Data Integrity Failures | Confiar en código o datos sin verificar su integridad | Esta lección |
| **A09** | Security Logging and Alerting Failures | No detectar ni alertar a tiempo un ataque | [Interpretación de logs](../01-autonomia-resolucion-problemas/03-interpretacion-de-logs.md) |
| **A10** | Mishandling of Exceptional Conditions | Errores mal manejados que dejan el sistema en un estado inseguro | Esta lección |

### 3. Cómo cambió desde 2017

| 2017 | 2021 | 2025 |
| --- | --- | --- |
| A1 Injection | A03 Injection (absorbe XSS) | **A05** Injection |
| A2 Broken Authentication | A07 Identification and Authentication Failures | **A07** Authentication Failures |
| A3 Sensitive Data Exposure | A02 Cryptographic Failures | **A04** Cryptographic Failures |
| A4 XML External Entities (XXE) | Absorbida por A05 Security Misconfiguration | Dentro de **A02** |
| A5 Broken Access Control | **A01** Broken Access Control | **A01** (absorbe SSRF) |
| A6 Security Misconfiguration | A05 Security Misconfiguration | **A02** |
| A7 Cross-Site Scripting (XSS) | Absorbida por A03 Injection | Dentro de **A05** |
| A8 Insecure Deserialization | Absorbida por A08 Software and Data Integrity Failures | Dentro de **A08** |
| A9 Using Components with Known Vulnerabilities | A06 Vulnerable and Outdated Components | **A03** Software Supply Chain Failures (más amplia) |
| A10 Insufficient Logging & Monitoring | A09 Security Logging and Monitoring Failures | **A09** Logging and **Alerting** Failures |
| — | A04 Insecure Design (nueva) | **A06** |
| — | A10 Server-Side Request Forgery (nueva) | Absorbida por **A01** |
| — | — | **A10** Mishandling of Exceptional Conditions (nueva) |

Lo que muestra la evolución:

* **XSS y XXE no desaparecieron**: se agruparon. XSS es un tipo de inyección; XXE, una mala configuración del parser.
* **El control de acceso es el problema número uno** desde 2021: es el más frecuente en las aplicaciones probadas.
* **La cadena de suministro creció**: ya no es solo "usar una versión vieja de una librería", sino también builds, pipelines y paquetes maliciosos.
* **Aparecen categorías de diseño y de manejo de errores**: no todo se resuelve con una función de escape; hay fallas que se introducen al diseñar.

### 4. A01: Broken Access Control

El usuario puede acceder a datos o ejecutar acciones fuera de lo que tiene permitido. El ejemplo clásico: el perfil de cada usuario está en `https://tienda.com/users/uber_haxor`, y cambiando la URL a `https://tienda.com/users/victima` se ve el perfil de otro. Nadie se preguntó si **la URL también es entrada del usuario**.

Incluye: modificar Ids para acceder a recursos ajenos (BOLA o IDOR), llamar endpoints de administración siendo un usuario común, elevar privilegios modificando un token, y desde 2025 también **SSRF**: lograr que el servidor pida recursos internos.

**Cómo se mitiga:** verificar en el servidor, en cada petición, que el usuario puede acceder a **ese** recurso concreto; denegar por defecto; y para SSRF, validar las URLs de destino contra una lista permitida. Se ve en detalle en [Control de acceso](03-control-de-acceso-bola-y-rbac.md).

### 5. A02: Security Misconfiguration

Todo el software de seguridad del mundo no sirve si está mal configurado. Ejemplos:

* almacenamiento en la nube (buckets) público por error;
* funciones innecesarias habilitadas: Swagger, páginas de diagnóstico o directorios listables en producción;
* mensajes de error detallados que revelan la tecnología, las rutas o el stack trace;
* cuentas o contraseñas por defecto;
* headers de seguridad faltantes (HSTS, CSP);
* parsers XML con entidades externas habilitadas (**XXE**).

**XXE**, en detalle. Un atacante sube un XML que declara una entidad apuntando a un archivo del servidor:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<methodCall>
  <methodName>&xxe;</methodName>
</methodCall>
```

Si el parser resuelve entidades externas, reemplaza `&xxe;` por el contenido de `/etc/passwd` (las cuentas del sistema en Linux), y puede devolverlo en la respuesta. En .NET moderno, `XmlReader` y `XmlDocument` **no** resuelven entidades externas por defecto (`DtdProcessing.Prohibit` y `XmlResolver = null`): el riesgo aparece si alguien las habilita o en código heredado de .NET Framework antiguo. La solución más simple sigue siendo: si puedes, no aceptes XML; usa JSON.

**Cómo se mitiga en general:** configuración segura por defecto, revisada en cada ambiente; nada de "configurar y olvidar". La configuración también es código: se versiona, se revisa en los PR y se audita.

### 6. A03: Software Supply Chain Failures

Con las vulnerabilidades conocidas, al atacante no le hace falta descubrir nada: el sistema **CVE** documenta públicamente cada vulnerabilidad, y existen herramientas que detectan qué versión de software corre un servidor y sugieren el exploit correspondiente.

En 2025 la categoría es más amplia que "usar una librería vieja":

| Riesgo | Ejemplo |
| --- | --- |
| Dependencias con vulnerabilidades conocidas | Una versión de Log4j afectada por Log4Shell (CVE-2021-44228) |
| Dependencias abandonadas | Un paquete sin mantenimiento hace años, que nadie va a parchear |
| Paquetes maliciosos | *Typosquatting*: `Newtonsoft.Jsonn` en lugar de `Newtonsoft.Json` |
| Pipeline comprometido | Un script de build descargado de internet sin verificar su integridad |
| Dependencias transitivas | La vulnerabilidad no está en tu paquete, sino en uno del que él depende |

**Cómo se mitiga:**

```text
dotnet list package --vulnerable --include-transitive   # .NET
npm audit                                               # Node
```

* análisis de dependencias automático en el pipeline (Dependabot, Renovate);
* actualizar con regularidad, no una vez al año;
* fijar versiones con archivos de bloqueo (`packages.lock.json`, `package-lock.json`);
* reemplazar dependencias abandonadas por alternativas mantenidas;
* mínimos permisos para los tokens del pipeline.

### 7. A04: Cryptographic Failures

Antes se llamaba "Sensitive Data Exposure"; se renombró porque la causa raíz suele ser criptográfica. Cuando hay una filtración, los datos robados se venden y revenden en la *dark web*, a menudo combinados en paquetes de identidad completos (*fullz*) que permiten cometer fraudes durante años.

Ejemplos: contraseñas guardadas con MD5 o en texto plano, datos que viajan por HTTP en lugar de HTTPS, claves de cifrado escritas en el código, algoritmos obsoletos. Se ve en [Autenticación y datos sensibles](05-autenticacion-y-datos-sensibles.md).

### 8. A05: Injection (incluye XSS)

Los datos del usuario terminan interpretados como **código** por un intérprete: SQL, el sistema operativo, LDAP o el navegador (XSS). El caso clásico, en una consulta armada concatenando texto:

```sql
-- el usuario escribe:  soap' UNION SELECT username, password, NULL FROM user_table;--
SELECT product_name, product_cost FROM product_table
WHERE product_name = 'soap' UNION SELECT username, password, NULL FROM user_table;--';
```

La entrada del usuario cerró la comilla y agregó su propia consulta. La causa no es "no haber limpiado la entrada", sino **mezclar datos y código en el mismo texto**. Se ve en detalle, con XSS, en [Inyección, XSS y sanitización](04-inyeccion-xss-y-sanitizacion.md).

### 9. A06: Insecure Design

Fallas que vienen del **diseño**, no de un error de implementación: aunque el código esté perfecto, la funcionalidad es insegura.

| Diseño inseguro | Por qué ningún código lo corrige |
| --- | --- |
| Recuperar la contraseña con una "pregunta secreta" ("¿nombre de tu mascota?") | La respuesta se puede averiguar en redes sociales |
| Un cupón de descuento reutilizable sin límite | El diseño no previó el abuso |
| Un flujo de compra donde el cliente envía el precio | El precio debería calcularse siempre en el servidor |
| Sin límite de intentos en un endpoint de verificación de códigos de 4 dígitos | 10 000 combinaciones se prueban en minutos |

**Cómo se mitiga:** modelado de amenazas antes de construir ([STRIDE](01-fundamentos-de-seguridad.md)), casos de abuso en las historias de usuario ("¿qué pasa si alguien intenta usar esto 10 000 veces?") y patrones de diseño seguros reutilizables.

### 10. A07: Authentication Failures

Vulnerabilidades que permiten suplantar a otro usuario. El ejemplo clásico: una cuenta de administrador "temporal" creada para el desarrollo, con usuario y contraseña `admin` / `admin`, que alguien olvidó borrar. La seguridad es tan fuerte como su eslabón más débil.

Incluye: credenciales por defecto o débiles, falta de límite de intentos (fuerza bruta y *credential stuffing*), sesiones que no expiran, tokens mal validados y falta de MFA. Se ve en [Autenticación y datos sensibles](05-autenticacion-y-datos-sensibles.md).

### 11. A08: Software or Data Integrity Failures

Confiar en código o datos sin verificar que no fueron alterados. Incluye la antigua **deserialización insegura**:

* **Serializar** es convertir un objeto en datos con formato (JSON, binario); **deserializar** es el camino inverso.
* Si un atacante controla los datos que se deserializan y el deserializador permite elegir **qué tipo** de objeto crear, puede instanciar clases que ejecutan código al construirse.

En .NET:

| Práctica | Riesgo |
| --- | --- |
| `BinaryFormatter` | Inseguro por diseño; en .NET 9 se eliminó su implementación y lanza una excepción |
| Newtonsoft.Json con `TypeNameHandling.All` o `Auto` | El JSON puede indicar el tipo a crear: permite ataques de deserialización |
| `System.Text.Json` deserializando a un DTO concreto | Seguro: solo crea el tipo que tú indicas |

Otros ejemplos: actualizaciones automáticas sin verificar su firma, cookies o tokens que el servidor confía sin validar la firma, y scripts cargados desde una CDN sin **Subresource Integrity** (`<script src="..." integrity="sha384-...">`).

**Cómo se mitiga:** no deserializar datos externos a tipos elegidos por el emisor; verificar firmas e integridad de todo lo que se ejecuta o se confía.

### 12. A09: Security Logging and Alerting Failures

En seguridad, el conocimiento es poder: la detección temprana marca la diferencia entre un memorando interno y una noticia que dura semanas. En 2025 la categoría cambió "monitoreo" por **alertas**: no alcanza con guardar logs si nadie se entera.

Compara dos logs del mismo ataque:

```text
# log pobre
Login failed
Login failed
Login failed
Login ok
```

```text
# log útil
2026-10-24T03:12:01.112Z WARN  auth.login.failed user=admin ip=203.0.113.45
2026-10-24T03:12:01.534Z WARN  auth.login.failed user=admin ip=203.0.113.45
... (7 más)
2026-10-24T03:12:04.902Z INFO  auth.login.ok     user=admin ip=203.0.113.45
2026-10-24T03:12:05.010Z INFO  auth.login.ok     user=ana   ip=198.51.100.7
```

Con el primero no se sabe si fueron usuarios distraídos o un ataque. Con el segundo es evidente: nueve intentos fallidos contra `admin` y un éxito en **cuatro segundos, desde la misma IP**: fuerza bruta automatizada. Y se distingue del login legítimo de `ana`.

**Cómo se mitiga:** logs estructurados de los eventos de seguridad (logins, cambios de permisos, accesos denegados, errores de validación), con fecha, usuario, IP y correlation ID (ver [Interpretación de logs](../01-autonomia-resolucion-problemas/03-interpretacion-de-logs.md)); **alertas** automáticas ante patrones sospechosos; y nunca datos sensibles en los logs.

### 13. A10: Mishandling of Exceptional Conditions

Nueva en 2025. Errores, condiciones inesperadas o estados anormales que el sistema no maneja bien, y que lo dejan inseguro.

```csharp
// Mal: falla ABIERTO
public async Task<bool> PuedeAccederAsync(int usuarioId, int recursoId)
{
    try
    {
        return await _permisos.VerificarAsync(usuarioId, recursoId);
    }
    catch (Exception)
    {
        return true; // "para no bloquear al usuario si se cae el servicio de permisos"
    }
}
```

Si el servicio de permisos se cae (o un atacante logra que falle), **todos** pueden acceder a todo.

| Fallar abierto (inseguro) | Fallar cerrado (seguro) |
| --- | --- |
| Si no se puede verificar el permiso, se permite | Si no se puede verificar el permiso, se niega |
| Si falla la validación del token, se continúa como anónimo con privilegios | Si falla la validación del token, `401` |
| Si una transacción falla a mitad, queda aplicada a medias | Si una transacción falla, se revierte completa |

Otros ejemplos: mensajes de error que revelan detalles internos (stack traces, consultas SQL) y excepciones que dejan recursos sin liberar. Se conecta con [Stack traces](../01-autonomia-resolucion-problemas/04-stack-traces.md) y con Problem Details en [Estándares RESTful](../03-diseno-de-apis-y-contratos/01-estandares-restful.md).

### 14. El OWASP API Security Top 10

Para APIs, OWASP publica una lista propia (edición 2023). Varias categorías coinciden con el Top 10 web, pero con nombres más específicos. Las que más importan para un backend:

| # | Categoría | Qué es |
| --- | --- | --- |
| API1 | **BOLA** (*Broken Object Level Authorization*) | Acceder a objetos ajenos cambiando su Id: `GET /orders/1043` |
| API2 | Broken Authentication | Autenticación débil o mal implementada |
| API3 | **BOPLA** (*Broken Object Property Level Authorization*) | Leer o modificar **propiedades** que no corresponden: la API devuelve `passwordHash`, o acepta `"isAdmin": true` en el body (asignación masiva) |
| API4 | Unrestricted Resource Consumption | Sin límites de tamaño, de cantidad ni de frecuencia de peticiones |
| API5 | **BFLA** (*Broken Function Level Authorization*) | Un usuario común llama a funciones de administración |
| API6 | Unrestricted Access to Sensitive Business Flows | Automatizar abusivamente un flujo legítimo (comprar todas las entradas con un bot) |
| API7 | Server-Side Request Forgery | El servidor pide URLs elegidas por el atacante |

BOLA es la **número uno** en APIs, por eso tiene su propia lección.

-----

## Ejemplo completo

Revisión de la API del problema con el Top 10 de 2025, endpoint por endpoint.

| Endpoint o componente | Hallazgo | Categoría | Corrección |
| --- | --- | --- | --- |
| `GET /api/orders/{id}` | Devuelve cualquier pedido, sea del usuario o no | A01 (BOLA) | Filtrar por el usuario autenticado |
| `GET /api/preview?url=` | Descarga cualquier URL, incluidas IPs internas | A01 (SSRF) | Lista de dominios permitidos; bloquear IPs privadas y de metadatos |
| `appsettings.Production.json` | `"DetailedErrors": true` y Swagger habilitado | A02 | Errores genéricos con Problem Details; Swagger solo en Development |
| Pipeline de CI | `curl https://install.algo.io \| bash` | A03 | Herramienta instalada desde un registro oficial, con versión fija |
| Paquetes | `SixLabors.ImageSharp` con un CVE conocido | A03 | Actualizar; Dependabot en el repositorio |
| Tabla `Users` | Contraseñas con SHA-256 sin salt | A04 | Hash con PBKDF2 o Argon2id (Identity) y migración en el próximo login |
| `/api/search` | `FromSqlRaw($"... LIKE '%{q}%'")` | A05 | `FromSql($"... LIKE {patron}")`, parametrizado |
| Cupones | Reutilizables sin límite por usuario | A06 | Regla de negocio: un uso por usuario, verificado en el servidor |
| Login | Sin límite de intentos; cuenta `admin` de prueba activa | A07 | Rate limiting y bloqueo temporal; eliminar la cuenta |
| Integración de pagos | Newtonsoft con `TypeNameHandling.Auto` en el webhook | A08 | `System.Text.Json` a un DTO concreto; verificar la firma del webhook |
| Logs | No registran los logins fallidos | A09 | Log estructurado de eventos de autenticación y alerta ante 10 fallos por minuto |
| Middleware de permisos | `catch` que devuelve `true` | A10 | Fallar cerrado: `catch` → `403` y log de error |

Qué observar:

* Las tres vulnerabilidades del problema (SSRF, pipeline y fallar abierto) solo aparecen con la lista actual.
* Cada hallazgo tiene una corrección concreta, no un "mejorar la seguridad".
* La mayoría de las correcciones son configuración o diseño, no código complejo.

-----

## Errores comunes

**1. Estudiar con una lista vieja.**
Qué pasa: se revisa XXE y se ignora SSRF, la cadena de suministro o el manejo de errores.
Por qué: el material de 2017 sigue circulando en cursos y tutoriales.
Arreglo: usa la edición vigente en owasp.org y fíjate en el año de cualquier material que leas.

**2. Tratar el Top 10 como un checklist completo.**
Qué pasa: "cumplimos el Top 10" y la aplicación igual tiene vulnerabilidades de lógica de negocio.
Por qué: se confunde un documento de concientización con un estándar.
Arreglo: el Top 10 para priorizar; el ASVS para requisitos verificables; modelado de amenazas para la lógica propia.

**3. Pensar que XSS "ya no importa" porque no está en la lista.**
Qué pasa: se baja la guardia con `dangerouslySetInnerHTML` o `Html.Raw`.
Por qué: XSS dejó de ser una categoría propia en 2021.
Arreglo: XSS sigue siendo inyección (A05): la misma regla de codificar la salida.

**4. Ignorar las dependencias transitivas.**
Qué pasa: tu paquete está actualizado, pero depende de otro vulnerable.
Por qué: solo se revisan las dependencias directas.
Arreglo: `--include-transitive` y escaneo automático en el pipeline.

**5. Fallar abierto "para no molestar al usuario".**
Qué pasa: un error en la verificación de permisos se convierte en acceso libre.
Por qué: se prioriza la disponibilidad sobre la confidencialidad sin pensarlo.
Arreglo: ante la duda, se niega; la indisponibilidad se resuelve con resiliencia, no saltándose controles.

-----

## Evolución y versiones

* **2001:** nace OWASP.
* **2003:** primera edición del Top 10.
* **2017:** la edición de tus apuntes: inyección en primer lugar; XXE e insecure deserialization como categorías propias.
* **2019:** primera edición del OWASP API Security Top 10.
* **2021:** Broken Access Control pasa al primer lugar; XSS se integra en Injection; aparecen Insecure Design y SSRF.
* **2023:** segunda edición del API Security Top 10, con BOLA en primer lugar.
* **2025:** aparecen Software Supply Chain Failures y Mishandling of Exceptional Conditions; SSRF se integra en Broken Access Control.

-----

## Cuándo sí y cuándo no

**Usa el Top 10 cuando:** quieres priorizar qué revisar en una aplicación, preparar un code review de seguridad, armar una capacitación o hablar un lenguaje común con un equipo de seguridad.

**Usa el ASVS cuando:** necesitas requisitos verificables para una auditoría o un contrato.

**No uses el Top 10 como:** la única fuente de seguridad, ni como prueba de que una aplicación es segura.

-----

## Resumen en 5 líneas

1. El Top 10 es un mapa de concientización basado en datos, no un checklist completo; la edición vigente es la de 2025.
2. A01 Broken Access Control es el riesgo número uno, e incluye ahora SSRF.
3. XSS y XXE no desaparecieron: XSS está en Injection (A05) y XXE en Security Misconfiguration (A02).
4. Novedades de 2025: cadena de suministro (A03) y manejo incorrecto de condiciones excepcionales (A10): fallar cerrado.
5. Para APIs existe el API Security Top 10, encabezado por BOLA.

-----

## Para profundizar

<details>
<summary>SSRF: por qué es tan grave en la nube</summary>

En AWS, Azure y GCP, cada máquina virtual puede consultar un servicio de metadatos en una IP interna (en AWS, `http://169.254.169.254/`), que entrega, entre otras cosas, credenciales temporales con los permisos de esa máquina. Si tu API descarga URLs que elige el usuario, un atacante puede pedirle que descargue esa dirección y devolverle las credenciales.

Defensas:

* lista de dominios permitidos, en lugar de bloquear algunos;
* resolver el DNS y rechazar IPs privadas, de *loopback* y de metadatos antes de conectar (y volver a verificar tras cada redirección);
* en AWS, exigir IMDSv2, que requiere un token y bloquea la mayoría de los ataques SSRF simples.

</details>

<details>
<summary>CWE, CVE y CVSS: cómo se nombra una vulnerabilidad</summary>

* **CWE** es el **tipo** de debilidad: CWE-89 es inyección SQL, en cualquier software.
* **CVE** es una vulnerabilidad **concreta** en un producto y versión concretos: CVE-2021-44228 es Log4Shell en Log4j 2.
* **CVSS** es el **puntaje de gravedad** de un CVE, de 0 a 10. Log4Shell tiene 10,0.

Cuando Dependabot avisa de un CVE con CVSS 9,8 en una dependencia, la prioridad es inmediata. Con un 3,1, se puede planificar.

</details>

-----

## En entrevista

### Respuesta corta (junior)

El OWASP Top 10 es una lista de las vulnerabilidades más comunes en aplicaciones web, como la inyección SQL, el XSS y el control de acceso roto. Sirve para saber qué revisar para hacer una aplicación más segura.

### Respuesta ampliada (semi-senior)

Es el documento de concientización de OWASP, basado en datos reales; la edición vigente es la de 2025. Lo uso para priorizar, sabiendo que no es un checklist completo. Destaco que el control de acceso roto es el primer riesgo desde 2021, y que en APIs se manifiesta sobre todo como BOLA, que verifico en cada endpoint filtrando por el usuario autenticado. Sé que XSS no desapareció, sino que está dentro de Injection, y que en 2025 aparecieron la cadena de suministro, que cubro con escaneo de dependencias en el pipeline, y el manejo de condiciones excepcionales, que se resume en fallar cerrado. Para APIs complemento con el API Security Top 10.

### Preguntas frecuentes de seguimiento

**1. ¿Cuál es el riesgo número uno y por qué?**
Broken Access Control, porque es el que más se encuentra en las aplicaciones probadas: es fácil de introducir (alcanza con olvidar una verificación en un endpoint) y difícil de detectar con herramientas automáticas.

**2. ¿Qué pasó con XSS?**
Se integró en la categoría Injection en 2021: es inyección de código en el navegador. Sigue siendo un riesgo vigente.

**3. ¿Qué significa fallar cerrado?**
Que si un control de seguridad no se puede evaluar por un error, el sistema niega el acceso en lugar de permitirlo.

-----

## Práctica

**Ejercicio 1.** Clasifica cada hallazgo según el Top 10 de 2025.

1. La API devuelve el stack trace completo cuando falla.
2. Un usuario cambia `/invoices/55` por `/invoices/56` y descarga una factura ajena.
3. El proyecto usa una versión de una librería JWT con un CVE crítico.
4. La búsqueda arma la consulta SQL concatenando el texto ingresado.
5. Los tokens de restablecimiento de contraseña nunca expiran.
6. Si Redis no responde, el rate limiter deja pasar todas las peticiones.

<details>
<summary>Solución</summary>

1. **A02** Security Misconfiguration (y relacionado con A10, por exponer detalles al fallar).
2. **A01** Broken Access Control (BOLA en el API Top 10).
3. **A03** Software Supply Chain Failures.
4. **A05** Injection.
5. **A07** Authentication Failures.
6. **A10** Mishandling of Exceptional Conditions: falla abierto. En el caso del rate limiter, a veces se acepta fallar abierto a propósito para no tumbar el servicio, pero tiene que ser una decisión consciente y monitoreada.

</details>

**Ejercicio 2.** Un compañero dice: "No usamos XML ni `BinaryFormatter`, así que de los viejos XXE e insecure deserialization estamos a salvo". ¿Qué le responderías?

<details>
<summary>Solución</summary>

Que es probable que tenga razón sobre esos dos casos puntuales, pero que las categorías actuales son más amplias:

* **A08** también incluye deserializar JSON con Newtonsoft y `TypeNameHandling` habilitado, confiar en tokens o webhooks sin verificar su firma, y scripts de CDN sin Subresource Integrity.
* **A02** incluye cualquier configuración insegura de parsers, servidores y nube, no solo XML.

La pregunta útil no es "¿usamos XML?", sino "¿confiamos en algún dato o código externo sin verificar su integridad?".

</details>

-----

## Siguiente lección

[Control de acceso: BOLA y RBAC](03-control-de-acceso-bola-y-rbac.md)
