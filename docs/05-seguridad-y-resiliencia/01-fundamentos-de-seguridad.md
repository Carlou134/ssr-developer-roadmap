# Fundamentos de seguridad web

## En una frase

La seguridad no es una funcionalidad que se agrega al final: es **proteger la confidencialidad, la integridad y la disponibilidad** de los datos desde el primer día, **asumiendo que alguien va a intentar atacar el sistema**, con varias capas de defensa, el mínimo privilegio necesario y controles automáticos en el pipeline (DevSecOps).

-----

## Antes de empezar

Conviene que ya sepas:

* Cómo funciona una API REST y sus códigos de estado: [Estándares RESTful](../03-diseno-de-apis-y-contratos/01-estandares-restful.md).
* Qué es la deuda técnica y el análisis estático: [Deuda técnica](../02-calidad-de-codigo-y-arquitectura/04-deuda-tecnica.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Vulnerabilidad:** debilidad del sistema que un atacante puede aprovechar.
* **Amenaza:** algo o alguien que podría aprovechar una vulnerabilidad.
* **Exploit:** la técnica o el código concreto que aprovecha una vulnerabilidad.
* **Superficie de ataque:** todos los puntos por los que alguien puede interactuar con el sistema (y, por lo tanto, atacarlo).
* **Tríada CIA:** confidencialidad, integridad y disponibilidad: los tres objetivos de la seguridad.
* **Defensa en profundidad:** varias capas de protección independientes, para que si una falla las demás sigan protegiendo.
* **Mínimo privilegio:** cada usuario, servicio o proceso tiene solo los permisos que necesita, nada más.
* **Modelado de amenazas:** analizar un diseño preguntando "¿qué puede salir mal?" antes de construirlo.
* **DevSecOps:** integrar controles de seguridad automáticos en todo el ciclo de desarrollo, no solo al final.
* **Pentesting (prueba de penetración):** simular un ataque real, con autorización, para encontrar vulnerabilidades antes que un atacante.

-----

## El problema

Una startup lanza su tienda online. La seguridad quedó "para después del lanzamiento". Tres meses más tarde:

* un usuario descubre que cambiando el número en `/api/orders/1043` puede ver los pedidos de otros clientes, con nombre, dirección y teléfono;
* la cuenta `admin` / `admin` que se creó para las pruebas sigue activa en producción;
* el archivo `appsettings.json` con la cadena de conexión a la base de datos estuvo público en GitHub durante dos meses;
* una dependencia de procesamiento de imágenes tiene una vulnerabilidad conocida desde hace un año;
* nadie lo notó a tiempo, porque los logs no registraban quién pedía qué.

Las consecuencias: datos de clientes expuestos, la obligación legal de notificar a los afectados, pérdida de confianza y semanas de trabajo para corregir lo que se podría haber evitado con prácticas básicas desde el día uno.

-----

## Cómo funciona

### 1. Por qué importa: qué puede hacer un atacante

Hoy todo está conectado: compras, bancos, hospitales y transporte dependen de aplicaciones web. Cuanto más abierto es el sistema, más oportunidades hay para un ataque. Si alguien compromete tu aplicación, puede:

| Ataque | Qué significa |
| --- | --- |
| **Robo de datos** | Extraer información personal, credenciales o tarjetas para venderla o usarla |
| **Defacement** | Cambiar el contenido visible del sitio |
| **Denegación de servicio (DoS)** | Dejar la aplicación inaccesible para los usuarios legítimos |
| **Toma de control** | Ejecutar código propio en tus servidores |
| **Pivote** | Usar tu sistema para atacar a otros (tus usuarios, tus proveedores) |

Y en todos los casos: pérdida de confianza de los usuarios, daño a la marca y, según el país, multas por las leyes de protección de datos.

### 2. La tríada CIA: los tres objetivos

| Principio | Qué protege | Ejemplos de controles |
| --- | --- | --- |
| **C**onfidencialidad | Que la información privada solo la vean quienes deben verla | Autenticación, autorización, cifrado, contraseñas con hash |
| **I**ntegridad | Que los datos no se alteren ni se borren sin permiso | Autorización, firmas criptográficas, validación, respaldos, auditoría |
| **A**vailability (disponibilidad) | Que el sistema esté accesible para quienes sí tienen permiso | Rate limiting, resiliencia, monitoreo, planes ante desastres |

Cada vulnerabilidad ataca al menos uno de los tres. Ver pedidos ajenos rompe la **confidencialidad**; modificarlos rompe la **integridad**; tumbar el sistema con miles de peticiones rompe la **disponibilidad**.

### 3. Seguridad proactiva: asume que te van a atacar

La seguridad es **proactiva**: hay que asumir que nada es seguro por defecto. Hay puntos de ataque en todas partes: el navegador, el HTML y el JavaScript, la API, las APIs externas que consumes, el protocolo HTTP, la base de datos, las dependencias y hasta el pipeline de despliegue.

Ejemplo: un formulario de login. Un atacante puede intentar:

* adivinar contraseñas por fuerza bruta (miles de intentos por minuto);
* probar credenciales filtradas de otros sitios (*credential stuffing*);
* inyectar SQL en el campo de usuario;
* robar la cookie de sesión de otro usuario.

Cada uno necesita su propia defensa. Por eso la regla de oro es: **no confíes en nada que venga de afuera**. El body, los query params, los headers, las cookies, **la URL** y los archivos subidos son entrada del usuario, aunque no haya un campo de texto a la vista.

### 4. Defensa en profundidad

Ningún control es perfecto. La **defensa en profundidad** pone varias capas independientes, de modo que si una falla, las demás siguen protegiendo.

```text
Usuario
  │
  ├─ 1. Red:            HTTPS, firewall, WAF, rate limiting
  ├─ 2. Autenticación:  ¿quién eres? (contraseña con hash, MFA, bloqueo por intentos)
  ├─ 3. Autorización:   ¿puedes hacer esto con ESTE recurso? (verificado en el servidor)
  ├─ 4. Validación:     ¿los datos tienen sentido? (tipos, rangos, formatos)
  ├─ 5. Código seguro:  consultas parametrizadas, salida codificada
  ├─ 6. Datos:          cifrado en reposo, mínimo privilegio en la base de datos
  └─ 7. Detección:      logs de seguridad, alertas, monitoreo
```

Por ejemplo: aunque tu código use consultas parametrizadas (capa 5), el usuario de la base de datos de la aplicación no debería tener permiso de `DROP TABLE` (capa 6). Si algún día aparece una inyección, el daño queda limitado.

### 5. Principios que se repiten en todo el resto de la fase

| Principio | Qué significa | Ejemplo |
| --- | --- | --- |
| **Mínimo privilegio** | Solo los permisos necesarios | El usuario de la base de datos de la API puede leer y escribir tablas, pero no crear ni borrar |
| **Denegar por defecto** | Si no hay una regla que lo permita, se niega | Todos los endpoints requieren autenticación salvo los marcados explícitamente como públicos |
| **Fallar de forma segura** | Si algo falla, el sistema queda en un estado seguro | Si el servicio de autorización no responde, se niega el acceso; no se permite "por si acaso" |
| **Seguro por defecto** | La configuración inicial es la segura | Sin cuentas de prueba, sin mensajes de error detallados en producción |
| **No seguridad por oscuridad** | El sistema es seguro aunque el atacante conozca cómo funciona | Un Id difícil de adivinar (GUID) **no** reemplaza la verificación de permisos |
| **Minimizar la superficie de ataque** | Menos endpoints, puertos y funciones expuestas, menos riesgos | Deshabilitar Swagger y endpoints de diagnóstico en producción |

### 6. DevSecOps: seguridad automática en el pipeline

Revisar la seguridad solo antes de un lanzamiento no escala: los problemas se descubren tarde, cuando son más caros de corregir. **DevSecOps** integra controles automáticos en cada etapa (lo que se llama *shift left*: mover la seguridad hacia el inicio del ciclo).

| Etapa | Control | Qué detecta | Herramientas |
| --- | --- | --- | --- |
| Mientras escribes | Analizadores en el IDE | Patrones inseguros (SQL concatenado, criptografía débil) | Analizadores de Roslyn, SonarLint |
| Antes de cada commit | Escaneo de secretos | Contraseñas, tokens y claves en el código | gitleaks, protección de push de GitHub |
| En cada PR | **SAST** (análisis estático) | Vulnerabilidades en el código fuente | SonarQube, CodeQL, Semgrep |
| En cada PR | **SCA** (análisis de dependencias) | Paquetes con vulnerabilidades conocidas (CVE) | Dependabot, `dotnet list package --vulnerable`, `npm audit` |
| En un entorno de prueba | **DAST** (análisis dinámico) | Vulnerabilidades en la aplicación en ejecución | OWASP ZAP, Burp Suite |
| Periódicamente | **Pentesting** | Lo que las herramientas automáticas no ven: lógica de negocio, combinaciones | Equipo especializado, con autorización |

El **pentesting** (también llamado *ethical hacking*) simula un ataque real para encontrar vulnerabilidades antes que un atacante. Requiere conocer sistemas, redes, protocolos, arquitectura y lenguajes de scripting. Como desarrollador no necesitas ser pentester, pero sí entender los ataques para no introducirlos.

> **Ojo:** probar ataques en sistemas sin autorización explícita es ilegal, aunque "solo estés aprendiendo". Practica en entornos hechos para eso: OWASP Juice Shop, DVWA, PortSwigger Web Security Academy o tus propios proyectos locales.

### 7. Modelado de amenazas: pensar como atacante antes de construir

Antes de construir una funcionalidad sensible, dedica unos minutos a preguntar "¿qué puede salir mal?". Una guía clásica es **STRIDE** (Microsoft):

| Letra | Amenaza | Principio que ataca | Pregunta |
| --- | --- | --- | --- |
| **S**poofing | Suplantar a otro | Autenticación | ¿Alguien puede hacerse pasar por otro usuario o servicio? |
| **T**ampering | Alterar datos | Integridad | ¿Alguien puede modificar datos en tránsito o en reposo? |
| **R**epudiation | Negar una acción | Auditoría | ¿Puede alguien negar que hizo algo, sin que podamos probarlo? |
| **I**nformation disclosure | Filtrar información | Confidencialidad | ¿Puede alguien ver datos que no le corresponden? |
| **D**enial of service | Dejar sin servicio | Disponibilidad | ¿Alguien puede dejar el sistema inutilizable? |
| **E**levation of privilege | Escalar privilegios | Autorización | ¿Puede un usuario común hacer algo de administrador? |

No hace falta un documento formal: una lista en el ticket de la funcionalidad, revisada con el equipo, ya cambia el diseño.

-----

## Ejemplo completo

Modelado de amenazas de la funcionalidad "ver el detalle de un pedido" (`GET /api/v1/orders/{id}`) de la tienda del problema.

| STRIDE | Amenaza concreta | Control | Lección |
| --- | --- | --- | --- |
| Spoofing | Usar la sesión robada de otro usuario | Cookies `HttpOnly` y `Secure`, tokens de vida corta | [Autenticación](05-autenticacion-y-datos-sensibles.md) |
| Tampering | Cambiar el `id` en la URL para ver otro pedido | Verificar en el servidor que el pedido pertenece al usuario | [Control de acceso](03-control-de-acceso-bola-y-rbac.md) |
| Repudiation | Un administrador modifica un pedido y lo niega | Log de auditoría con usuario, acción, fecha e IP | [Interpretación de logs](../01-autonomia-resolucion-problemas/03-interpretacion-de-logs.md) |
| Information disclosure | La respuesta incluye datos internos o un stack trace | DTO de salida; Problem Details sin detalles en producción | [Estándares RESTful](../03-diseno-de-apis-y-contratos/01-estandares-restful.md) |
| Denial of service | Un script pide miles de pedidos por minuto | Rate limiting por usuario | [Rate limiting](08-rate-limiting.md) |
| Elevation of privilege | Un usuario común llama al endpoint de administración | Políticas de autorización por rol | [Control de acceso](03-control-de-acceso-bola-y-rbac.md) |

Y en el pipeline del repositorio:

```yaml
# .github/workflows/seguridad.yml (resumen)
on: [pull_request]
jobs:
  seguridad:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: Secretos en el historial
        uses: gitleaks/gitleaks-action@v2
      - name: Dependencias vulnerables
        run: dotnet list package --vulnerable --include-transitive
      - name: Análisis estático
        uses: github/codeql-action/analyze@v3
```

Qué observar:

* Cada amenaza del problema (pedidos ajenos, cuenta de prueba, secretos públicos, dependencia vulnerable, logs ciegos) tiene un control concreto y una lección de esta fase.
* El modelado de amenazas se hizo **antes** de programar el endpoint, en minutos, con una tabla.
* El pipeline frena el PR si aparece un secreto o una dependencia vulnerable: la seguridad no depende de que alguien se acuerde.

-----

## Errores comunes

**1. "La seguridad la vemos después".**
Qué pasa: vulnerabilidades en producción que obligan a rediseñar partes del sistema.
Por qué: la seguridad se trata como una funcionalidad opcional y no como un requisito.
Arreglo: modelado de amenazas al diseñar y controles automáticos en el pipeline desde el primer día.

**2. Confiar en lo que viene del cliente.**
Qué pasa: precios, Ids de usuario o roles manipulados desde el navegador.
Por qué: se asume que el frontend "solo envía lo que mostramos".
Arreglo: todo lo que llega del cliente (body, URL, headers, cookies) es entrada no confiable y se valida en el servidor.

**3. Seguridad por oscuridad.**
Qué pasa: endpoints "ocultos" o Ids "imposibles de adivinar" que alguien termina encontrando.
Por qué: se reemplaza un control real por la esperanza de que nadie lo descubra.
Arreglo: verificación de autenticación y autorización en cada endpoint, se conozca o no su URL.

**4. Una sola capa de defensa.**
Qué pasa: un solo error (una consulta concatenada) compromete todo el sistema.
Por qué: el usuario de la base de datos es administrador, no hay rate limiting ni alertas.
Arreglo: defensa en profundidad y mínimo privilegio en cada capa.

**5. Probar ataques sin autorización.**
Qué pasa: consecuencias legales, aunque la intención fuera aprender.
Por qué: se prueba "solo un `' OR 1=1 --`" en un sitio real.
Arreglo: practica en entornos diseñados para eso o en tus propios proyectos.

-----

## Cuándo sí y cuándo no

**Haz modelado de amenazas cuando:** la funcionalidad maneja datos personales, dinero, permisos o autenticación, o expone algo nuevo a internet.

**Integra controles en el pipeline siempre:** escaneo de secretos y de dependencias cuesta minutos de configurar y funciona solo.

**No hace falta un modelado formal cuando:** el cambio no toca datos, permisos ni nuevas superficies de ataque (un cambio de estilos, un texto). Pero la regla "no confíes en la entrada" aplica siempre.

-----

## Resumen en 5 líneas

1. La seguridad protege la confidencialidad, la integridad y la disponibilidad (tríada CIA).
2. Asume que te van a atacar: todo lo que viene de afuera, incluida la URL, es entrada no confiable.
3. Defensa en profundidad, mínimo privilegio, denegar por defecto y fallar de forma segura.
4. Modelado de amenazas con STRIDE antes de construir lo sensible.
5. DevSecOps: escaneo de secretos, SAST, SCA y DAST automáticos en el pipeline; pentesting periódico.

-----

## Para profundizar

<details>
<summary>Leyes de protección de datos: por qué la seguridad también es legal</summary>

Muchos países tienen leyes que obligan a proteger los datos personales y a notificar las filtraciones: el GDPR en la Unión Europea, la LGPD en Brasil, la Ley 29733 en Perú, la Ley 25.326 en Argentina, entre otras. En general exigen:

* recolectar solo los datos necesarios (minimización);
* protegerlos con medidas técnicas razonables (cifrado, control de acceso);
* notificar a las autoridades y a los afectados si hay una filtración, en plazos cortos.

Como desarrollador, el impacto práctico es: **no guardes datos que no necesitas**, y trata los datos personales como sensibles en logs, respaldos y entornos de prueba.

</details>

<details>
<summary>Dónde practicar de forma legal</summary>

* **OWASP Juice Shop:** una tienda deliberadamente vulnerable, con desafíos para cada categoría del Top 10. Se levanta con `docker run -p 3000:3000 bkimminich/juice-shop`.
* **PortSwigger Web Security Academy:** laboratorios gratuitos y guiados para cada tipo de vulnerabilidad.
* **DVWA (Damn Vulnerable Web Application):** aplicación PHP vulnerable con niveles de dificultad.

Atacar tu propia instancia local de estas aplicaciones es la mejor forma de entender por qué existe cada defensa.

</details>

-----

## En entrevista

### Respuesta corta (junior)

La seguridad web protege los datos y el sistema de accesos no autorizados. Se basa en la tríada CIA: confidencialidad, integridad y disponibilidad. Lo principal es no confiar en la entrada del usuario y validar todo en el servidor.

### Respuesta ampliada (semi-senior)

Trato la seguridad como un requisito desde el diseño, no como algo que se agrega al final. Asumo que todo lo que llega del cliente, incluida la URL, es hostil, y aplico defensa en profundidad: HTTPS y rate limiting, autenticación robusta, autorización verificada en el servidor por recurso, consultas parametrizadas, mínimo privilegio en la base de datos y logs de seguridad con alertas. Para funcionalidades sensibles hago un modelado de amenazas rápido con STRIDE. Y lo automatizo: escaneo de secretos, SAST y análisis de dependencias en cada PR, para que la seguridad no dependa de la memoria de nadie.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es la defensa en profundidad?**
Varias capas de protección independientes, para que si una falla las demás sigan protegiendo. Por ejemplo, consultas parametrizadas y, además, un usuario de base de datos sin permisos de borrar tablas.

**2. ¿Qué diferencia hay entre SAST y DAST?**
SAST analiza el código fuente sin ejecutarlo; DAST ataca la aplicación en ejecución desde afuera, como lo haría un atacante. Son complementarios.

**3. ¿Por qué un GUID no es una medida de seguridad?**
Porque dificulta adivinar un Id, pero no impide que alguien que lo obtenga (de un log, un link compartido, una respuesta) acceda al recurso. La protección real es verificar los permisos en el servidor.

-----

## Práctica

**Ejercicio 1.** Clasifica cada incidente según el principio de la tríada CIA que rompe.

1. Un empleado descarga la lista de clientes con sus tarjetas.
2. Un atacante cambia el precio de un producto a 0,01.
3. Un ataque de miles de peticiones por segundo deja la tienda caída durante el Black Friday.
4. Un usuario borra los comentarios de otros usuarios.
5. Un log de producción guarda las contraseñas en texto plano.

<details>
<summary>Solución</summary>

1. **Confidencialidad.**
2. **Integridad.**
3. **Disponibilidad.**
4. **Integridad** (y además, un problema de autorización).
5. **Confidencialidad:** cualquiera con acceso a los logs ve las contraseñas.

</details>

**Ejercicio 2.** Aplica STRIDE a la funcionalidad "cambiar el email de mi cuenta" (`PUT /api/v1/me/email`). Encuentra al menos cuatro amenazas y su control.

<details>
<summary>Solución</summary>

| STRIDE | Amenaza | Control |
| --- | --- | --- |
| Spoofing | Alguien con la sesión abierta en una computadora compartida cambia el email y se queda con la cuenta | Pedir la contraseña actual (reautenticación) para cambios sensibles |
| Tampering | Cambiar el Id de usuario en el body para modificar el email de otro | Tomar el usuario de la sesión o del token, nunca del body |
| Repudiation | El dueño dice que nunca cambió el email | Log de auditoría y aviso al email **anterior** |
| Information disclosure | La respuesta revela si el nuevo email ya existe en el sistema | Mensaje genérico; confirmación por correo al nuevo email |
| Denial of service | Un script cambia el email miles de veces para enviar spam con los correos de confirmación | Rate limiting por usuario |
| Elevation of privilege | Agregar `"role": "admin"` en el body | DTO que solo acepta `email` (sin asignación masiva) |

</details>

-----

## Siguiente lección

[OWASP Top 10 (2025)](02-owasp-top-10.md)
