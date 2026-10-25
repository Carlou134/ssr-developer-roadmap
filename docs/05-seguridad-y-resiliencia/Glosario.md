# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**ABAC (*Attribute-Based Access Control*).** Permisos según atributos del usuario, del recurso o del contexto: dueño, sucursal, horario.

**Ambiente (*environment*).** Cada lugar donde corre la aplicación: desarrollo, pruebas, *staging*, producción.

**Amenaza.** Algo o alguien que podría aprovechar una vulnerabilidad.

**Asignación masiva (*mass assignment*).** El servidor copia al objeto todos los campos que envía el cliente, incluidos los que no debería poder modificar, como el precio o el rol.

**Autenticación (AuthN).** Verificar la identidad: ¿quién eres?

**Autorización (AuthZ).** Verificar los permisos: ¿puedes hacer esto con este recurso?

**Autorización basada en recursos.** Decidir el permiso según el recurso concreto (su dueño, su estado), no solo según el usuario.

**BFLA (*Broken Function Level Authorization*).** Un usuario ejecuta una función (un endpoint) que no corresponde a su rol, como una acción de administrador.

**BOLA (*Broken Object Level Authorization*).** Acceder a un objeto ajeno cambiando su identificador. También se llama IDOR. Es el riesgo número uno del OWASP API Security Top 10.

**BOPLA (*Broken Object Property Level Authorization*).** Leer o modificar propiedades de un objeto que no corresponden: exponer `passwordHash` o aceptar `isAdmin` en el body.

**Cadena de suministro (*supply chain*).** Todo lo que interviene en construir y distribuir el software: dependencias, herramientas de build, pipelines, registros de paquetes.

**Circuit breaker (cortocircuito).** Mecanismo que, tras muchas fallas, deja de llamar a un servicio durante un tiempo y falla de inmediato. Tiene tres estados: cerrado, abierto y semiabierto.

**Cifrado.** Transformación reversible con una clave; quien tiene la clave recupera el dato original.

**Claim.** Un dato sobre el usuario incluido en su identidad, como su Id, su email o sus roles.

**Codificación de salida (*output encoding*).** Convertir caracteres especiales (`<`, `>`, `"`) en su forma segura (`&lt;`, `&gt;`, `&quot;`) para que se muestren como texto.

**Consulta parametrizada.** La consulta y los datos viajan por separado al motor; los datos nunca se interpretan como SQL.

**CSP (*Content Security Policy*).** Header que le indica al navegador de qué orígenes puede cargar y ejecutar scripts.

**CSRF (*Cross-Site Request Forgery*).** Lograr que el navegador de la víctima envíe una petición a tu sitio, con sus cookies, sin que lo sepa.

**Credential stuffing.** Probar combinaciones de usuario y contraseña filtradas de otros sitios.

**CVE (*Common Vulnerabilities and Exposures*).** Identificador público de una vulnerabilidad concreta en un producto concreto.

**CWE (*Common Weakness Enumeration*).** Catálogo de tipos de debilidades de software.

**Defensa en profundidad.** Varias capas de protección independientes, para que si una falla las demás sigan protegiendo.

**Deserialización insegura.** Convertir datos no confiables en objetos de forma que el atacante controle qué se crea o se ejecuta.

**DevSecOps.** Integrar controles de seguridad automáticos en todo el ciclo de desarrollo, no solo al final.

**Exploit.** La técnica o el código concreto que aprovecha una vulnerabilidad.

**Fail open / fail closed (fallar abierto / cerrado).** Ante un error, el sistema permite el acceso (abierto, inseguro) o lo niega (cerrado, seguro).

**Falla transitoria.** Falla temporal que probablemente se resuelva sola: un timeout, un `503`, una conexión cortada.

**Fallas en cascada.** Un servicio lento o caído arrastra a los que dependen de él.

**Fallback.** Respuesta alternativa cuando una operación falla: un valor en caché, un valor por defecto.

**Fuerza bruta.** Probar muchas contraseñas hasta acertar.

**Gestor de secretos (*secrets manager*).** Servicio que guarda, cifra, controla el acceso y audita los secretos: Azure Key Vault, AWS Secrets Manager, HashiCorp Vault.

**Hash.** Función de un solo sentido que convierte un dato en una huella de longitud fija; no se puede revertir.

**HSTS (*HTTP Strict Transport Security*).** Header que obliga al navegador a usar siempre HTTPS con tu dominio.

**Identidad administrada (*managed identity*).** Identidad que la nube asigna a tu aplicación para que acceda a otros servicios sin una contraseña guardada.

**Inyección.** Datos del usuario que terminan interpretados como código por un intérprete: SQL, el shell, el navegador.

**Jitter.** Variación aleatoria en la espera entre reintentos, para que muchos clientes no reintenten al mismo tiempo.

**JWT (*JSON Web Token*).** Token firmado (no cifrado) que contiene claims del usuario en formato JSON.

**KDF (función de derivación de claves).** Función de hash lenta a propósito, diseñada para contraseñas: PBKDF2, bcrypt, scrypt, Argon2.

**MFA (*Multi-Factor Authentication*).** Exigir dos o más factores de autenticación: algo que sabes, algo que tienes, algo que eres.

**Mínimo privilegio.** Cada usuario, servicio o proceso tiene solo los permisos que necesita.

**Modelado de amenazas.** Analizar un diseño preguntando "¿qué puede salir mal?" antes de construirlo.

**OWASP (*Open Worldwide Application Security Project*).** Fundación que publica guías, herramientas y estándares de seguridad de aplicaciones, entre ellos el Top 10.

**Partición (de rate limiting).** El criterio con el que se agrupan las peticiones para contarlas: por usuario, IP o API key.

**Pentesting (prueba de penetración).** Simular un ataque real, con autorización, para encontrar vulnerabilidades antes que un atacante.

**PII (*Personally Identifiable Information*).** Datos que identifican a una persona: nombre, documento, email, dirección, teléfono.

**Pipeline de resiliencia.** Combinación ordenada de estrategias (timeout, retry, circuit breaker) que envuelve una operación.

**Polly.** La librería de resiliencia más usada en .NET.

**Proxy inverso.** Servidor que recibe las peticiones y las reenvía a tu aplicación: un balanceador, Nginx, un API Gateway.

**Rate limiting (limitación de tasa).** Limitar cuántas peticiones acepta el servidor por cliente en un período.

**RBAC (*Role-Based Access Control*).** Permisos asignados según el rol del usuario: administrador, vendedor, cliente.

**Refresh token.** Token de vida larga que solo sirve para obtener nuevos tokens de acceso.

**Resiliencia.** Capacidad de un sistema de seguir funcionando cuando algo falla, y de recuperarse.

**Retry (reintento).** Volver a ejecutar una operación que falló.

**Rotación (de secretos).** Reemplazar un secreto por uno nuevo y revocar el anterior.

**Salt.** Valor aleatorio y único que se agrega a cada contraseña antes de calcular su hash.

**Sanitización.** Limpiar una entrada que tiene que contener HTML, dejando solo lo permitido.

**Secreto.** Dato que da acceso a algo: contraseña, cadena de conexión, API key, clave privada, token.

**SSRF (*Server-Side Request Forgery*).** Lograr que el servidor haga peticiones a destinos elegidos por el atacante, como servicios internos o los metadatos de la nube.

**STRIDE.** Guía de modelado de amenazas: suplantación, alteración, repudio, filtración de información, denegación de servicio y elevación de privilegios.

**Superficie de ataque.** Todos los puntos por los que alguien puede interactuar con el sistema y, por lo tanto, atacarlo.

**Timeout.** Tiempo máximo de espera antes de abandonar una operación.

**Token bucket.** Algoritmo de rate limiting con un balde que se recarga a ritmo constante; cada petición consume un token.

**Tormenta de reintentos (*retry storm*).** Los reintentos de muchos clientes o capas multiplican la carga sobre un servicio ya sobrecargado.

**Tríada CIA.** Confidencialidad, integridad y disponibilidad: los tres objetivos de la seguridad.

**User Secrets.** Almacén de secretos de .NET para desarrollo, guardado fuera del repositorio, en el perfil del usuario.

**Validación.** Rechazar la entrada que no cumple el formato esperado.

**Ventana fija / deslizante.** Algoritmos de rate limiting que cuentan las peticiones en intervalos fijos o en los últimos N segundos.

**Vulnerabilidad.** Debilidad del sistema que un atacante puede aprovechar.

**XSS (*Cross-Site Scripting*).** JavaScript del atacante que se ejecuta en el navegador de otro usuario, dentro de tu sitio. Puede ser reflejado, almacenado o basado en DOM.

**XXE (*XML External Entities*).** Abuso de un parser XML para leer archivos internos o hacer peticiones.
