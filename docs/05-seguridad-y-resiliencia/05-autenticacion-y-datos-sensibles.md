# Autenticación y datos sensibles

## En una frase

Autenticar bien es **no inventar nada**: contraseñas guardadas con un hash lento y con salt (nunca cifradas ni en texto plano), límite de intentos y MFA contra la fuerza bruta, tokens y cookies validados y de vida corta, y datos sensibles cifrados en tránsito (HTTPS) y en reposo, sin aparecer jamás en logs ni en respuestas.

-----

## Antes de empezar

Conviene que ya sepas:

* La diferencia entre autenticación y autorización: [Control de acceso](03-control-de-acceso-bola-y-rbac.md).
* Qué es XSS y por qué permite robar datos del navegador: [Inyección, XSS y sanitización](04-inyeccion-xss-y-sanitizacion.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Hash:** función de un solo sentido que convierte un dato en una huella de longitud fija; no se puede revertir.
* **Salt:** valor aleatorio y único que se agrega a cada contraseña antes de calcular su hash.
* **Función de derivación de claves (KDF):** función de hash **lenta** a propósito, diseñada para contraseñas: PBKDF2, bcrypt, scrypt, Argon2.
* **Cifrado:** transformación reversible con una clave; quien tiene la clave recupera el dato original.
* **Fuerza bruta:** probar muchas contraseñas hasta acertar.
* **Credential stuffing:** probar combinaciones de usuario y contraseña filtradas de otros sitios.
* **MFA (*Multi-Factor Authentication*):** exigir dos o más factores: algo que sabes, algo que tienes, algo que eres.
* **JWT (*JSON Web Token*):** token firmado que contiene claims del usuario en formato JSON.
* **Refresh token:** token de vida larga que solo sirve para obtener nuevos tokens de acceso.
* **CSRF (*Cross-Site Request Forgery*):** lograr que el navegador de la víctima envíe una petición a tu sitio, con sus cookies, sin que lo sepa.
* **HSTS (*HTTP Strict Transport Security*):** header que obliga al navegador a usar siempre HTTPS con tu dominio.
* **PII (*Personally Identifiable Information*):** datos que identifican a una persona: nombre, documento, email, dirección, teléfono.

-----

## El problema

El sistema de usuarios de la tienda funciona así:

```csharp
// registro
usuario.PasswordHash = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(password)));

// login: sin límite de intentos
var u = await db.Users.FirstOrDefaultAsync(x => x.Email == email);
if (u is null) return Unauthorized("El email no existe");
if (u.PasswordHash != Hash(password)) return Unauthorized("Contraseña incorrecta");

var token = GenerarJwt(u, expira: TimeSpan.FromDays(30));
return Ok(new { token }); // el frontend lo guarda en localStorage
```

Además, el log de errores registra el body completo de cada petición fallida, y el sitio todavía acepta HTTP.

Las consecuencias:

* la base se filtra y en horas se recupera el 60 % de las contraseñas: SHA-256 sin salt es rapidísimo, y las contraseñas comunes ya tienen su hash en tablas precalculadas;
* "El email no existe" frente a "Contraseña incorrecta" permite saber quién está registrado (**enumeración de usuarios**);
* sin límite de intentos, un script prueba miles de contraseñas por minuto;
* un XSS en cualquier página lee el token de `localStorage`, que sirve durante 30 días y no se puede revocar;
* el log de errores contiene contraseñas en texto plano;
* en una red pública, el login viaja por HTTP y cualquiera en la misma red lo puede leer.

-----

## Cómo funciona

### 1. Hash frente a cifrado: las contraseñas no se cifran

| | Cifrado | Hash |
| --- | --- | --- |
| ¿Es reversible? | Sí, con la clave | No |
| Uso | Datos que necesitas **leer** después (un número de tarjeta para cobrar) | Datos que solo necesitas **comparar** (una contraseña) |
| Si roban la base | Si también roban la clave, recuperan todo | Tienen que adivinar cada contraseña |

Una contraseña nunca necesita leerse: solo se compara. Por eso **se guarda con hash, nunca cifrada ni en texto plano**. En el login, se calcula el hash de lo que escribió el usuario y se compara con el guardado.

### 2. No cualquier hash: lento y con salt

SHA-256 es un excelente hash para verificar integridad, pero **terrible para contraseñas**, por dos razones:

| Problema | Por qué | Solución |
| --- | --- | --- |
| Es **rápido** | Una GPU calcula miles de millones de SHA-256 por segundo: adivinar contraseñas comunes es inmediato | Una **KDF lenta** a propósito, con un costo configurable |
| **Sin salt**, la misma contraseña da el mismo hash | Existen tablas precalculadas (*rainbow tables*) con los hashes de millones de contraseñas; y dos usuarios con `123456` tienen el mismo hash | Un **salt aleatorio por usuario**, guardado junto al hash |

Las funciones recomendadas por OWASP para contraseñas:

| Función | Comentario |
| --- | --- |
| **Argon2id** | La primera opción recomendada; resistente a ataques con GPU por su uso de memoria |
| **scrypt** | Buena alternativa, también exigente en memoria |
| **bcrypt** | Muy usada y madura; limita las contraseñas a 72 bytes |
| **PBKDF2** | Aceptable con muchas iteraciones (OWASP sugiere 600 000 con SHA-256); requerida en entornos con certificación FIPS |

**En ASP.NET Core no lo implementes a mano:** ASP.NET Core Identity ya trae `PasswordHasher<TUser>`, que usa PBKDF2 con salt aleatorio y guarda en el mismo valor la versión del algoritmo, el salt y el hash. Desde .NET 7 usa HMAC-SHA512 con 100 000 iteraciones.

```csharp
var hasher = new PasswordHasher<Usuario>();

string hash = hasher.HashPassword(usuario, password); // al registrar

var resultado = hasher.VerifyHashedPassword(usuario, usuario.PasswordHash, passwordIngresado);
// Success, Failed o SuccessRehashNeeded (el hash es de una versión vieja: recalcúlalo y guárdalo)
```

`SuccessRehashNeeded` permite migrar los hashes viejos de forma transparente: en el siguiente login exitoso, se recalcula con los parámetros actuales.

### 3. Defensas contra la fuerza bruta y el credential stuffing

| Ataque | Defensa |
| --- | --- |
| Muchos intentos contra una cuenta | Bloqueo temporal tras N intentos fallidos (Identity: `Lockout`) |
| Muchos intentos desde una IP | [Rate limiting](08-rate-limiting.md) en el endpoint de login |
| Contraseñas filtradas de otros sitios | MFA; rechazar contraseñas conocidas como filtradas (por ejemplo, con la API de Have I Been Pwned) |
| Contraseñas débiles | Largo mínimo (al menos 8, mejor 12 o más) en lugar de reglas de complejidad raras |
| Cuentas por defecto | Ninguna cuenta `admin` / `admin`; las cuentas de prueba no llegan a producción |

Sobre las contraseñas, las recomendaciones actuales (NIST SP 800-63B) priorizan el **largo** y la **comparación contra listas de contraseñas filtradas**, y desaconsejan obligar a cambiarlas periódicamente sin motivo o exigir combinaciones de símbolos que llevan a contraseñas predecibles como `Verano2026!`.

**Mensajes genéricos** para no revelar qué cuentas existen:

```text
Mal:   "El email no existe"  /  "Contraseña incorrecta"
Bien:  "Email o contraseña incorrectos"
```

Lo mismo en "olvidé mi contraseña": responde siempre "Si el email está registrado, te enviamos un enlace", exista o no.

### 4. Sesiones con cookies frente a tokens (JWT)

Después de autenticar, el servidor necesita reconocer al usuario en cada petición. Hay dos enfoques:

| | Cookie de sesión | Token (JWT) |
| --- | --- | --- |
| Dónde vive el estado | En el servidor (o cifrado dentro de la cookie) | Dentro del token, firmado |
| Cómo viaja | El navegador envía la cookie automáticamente | El cliente lo envía en `Authorization: Bearer ...` |
| Revocar una sesión | Simple: se borra en el servidor | Difícil: es válido hasta que expira (salvo listas de revocación) |
| Ideal para | Aplicaciones web del mismo dominio | APIs consumidas por apps móviles, otros servicios o varios dominios |
| Riesgo principal | CSRF (mitigado con `SameSite`) | Robo del token por XSS si se guarda en JavaScript |

Ninguno es "más seguro" en abstracto: depende de dónde se guarda y cómo se valida.

### 5. Cookies seguras

```csharp
builder.Services.ConfigureApplicationCookie(o =>
{
    o.Cookie.HttpOnly = true;                              // JavaScript no puede leerla: un XSS no la roba
    o.Cookie.SecurePolicy = CookieSecurePolicy.Always;     // solo viaja por HTTPS
    o.Cookie.SameSite = SameSiteMode.Lax;                  // no se envía en peticiones cruzadas (POST) desde otros sitios
    o.ExpireTimeSpan = TimeSpan.FromHours(8);
    o.SlidingExpiration = true;
});
```

| Atributo | Protege contra |
| --- | --- |
| `HttpOnly` | Robo de la cookie por XSS (`document.cookie` no la ve) |
| `Secure` | Que viaje por HTTP y alguien la lea en la red |
| `SameSite=Lax` o `Strict` | CSRF: el navegador no la envía en peticiones originadas en otros sitios |

**CSRF**, en una frase: si tu sitio usa cookies y un sitio malicioso tiene un formulario que hace `POST` a `https://tienda.com/api/transferir`, el navegador de la víctima envía la petición **con sus cookies**. `SameSite=Lax` lo bloquea para los `POST` entre sitios; para aplicaciones con formularios clásicos, ASP.NET Core agrega además tokens antifalsificación (*antiforgery*).

### 6. JWT: lo que hay que saber

Un JWT tiene tres partes separadas por puntos: `encabezado.payload.firma`, codificadas en Base64URL.

```text
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI0MiIsInJvbGUiOiJDbGllbnRlIiwiZXhwIjoxNzYxMzQwMDAwfQ.k3x...
```

```json
{ "sub": "42", "role": "Cliente", "exp": 1761340000 }
```

> **Ojo: un JWT está firmado, no cifrado.** Cualquiera puede decodificar el payload (pega el token en jwt.io y lo verás). La firma garantiza que **no fue modificado**, no que sea secreto. **Nunca** pongas datos sensibles en un JWT.

Reglas para usarlo bien:

| Regla | Por qué |
| --- | --- |
| Validar firma, emisor (`iss`), audiencia (`aud`) y expiración (`exp`) | Un token de otro sistema o vencido no debe aceptarse |
| Fijar el algoritmo esperado | Evita el ataque `"alg": "none"` y la confusión de algoritmos |
| Tokens de acceso de vida **corta** (5 a 15 minutos) | Si lo roban, sirve poco tiempo, porque no se puede revocar fácilmente |
| **Refresh token** para renovar | De vida larga, guardado de forma segura, rotado en cada uso y revocable en el servidor |
| No guardarlo en `localStorage` en aplicaciones web | Cualquier XSS lo lee; mejor una cookie `HttpOnly` o el patrón BFF |

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o =>
    {
        o.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "https://auth.tienda.com",
            ValidateAudience = true,
            ValidAudience = "api-tienda",
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromSeconds(30), // por defecto son 5 minutos de tolerancia
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(clave)), // la clave viene de un secreto
            ValidAlgorithms = [SecurityAlgorithms.HmacSha256]
        };
    });
```

> **Consejo:** no construyas tu propio servidor de identidad salvo que sea imprescindible. Login, MFA, recuperación de contraseña, refresh tokens y revocación tienen muchos detalles. Un proveedor (ASP.NET Core Identity, Keycloak, Auth0, Microsoft Entra ID, Cognito) usando OpenID Connect ya los resuelve.

### 7. Datos sensibles: en tránsito, en reposo y en los logs

**En tránsito: HTTPS siempre.**

```csharp
app.UseHsts();             // el navegador solo usará HTTPS con tu dominio
app.UseHttpsRedirection(); // HTTP → HTTPS
```

**En reposo: cifrar lo que hace falta leer.** Un número de documento o los datos de una tarjeta, si de verdad necesitas guardarlos, van cifrados. Mejor todavía: no los guardes (para tarjetas, un proveedor de pagos tokeniza el número y tú solo guardas un token).

En .NET, la **Data Protection API** cifra datos con claves que rota automáticamente:

```csharp
public class DocumentoProtegido(IDataProtectionProvider proveedor)
{
    private readonly IDataProtector _protector = proveedor.CreateProtector("Clientes.Documento");

    public string Cifrar(string documento) => _protector.Protect(documento);
    public string Descifrar(string cifrado) => _protector.Unprotect(cifrado);
}
```

**Minimización:** el dato más seguro es el que no guardas. Antes de agregar un campo sensible, pregunta si es imprescindible.

**En logs y respuestas: nunca.**

| Nunca en un log | Nunca en una respuesta de la API |
| --- | --- |
| Contraseñas, tokens, refresh tokens | `PasswordHash`, `SecurityStamp` |
| Números de tarjeta, CVV | Datos internos (`CostoInterno`) |
| Documentos completos, datos de salud | Stack traces o consultas SQL en producción |

Se conecta con los DTOs de salida ([Arquitectura en capas](../02-calidad-de-codigo-y-arquitectura/08-arquitectura-en-capas-ntier.md)) y con el logging estructurado ([Interpretación de logs](../01-autonomia-resolucion-problemas/03-interpretacion-de-logs.md)).

-----

## Ejemplo completo

El login del problema, con ASP.NET Core Identity.

`Program.cs`:

```csharp
builder.Services.AddIdentityCore<Usuario>(o =>
{
    o.Password.RequiredLength = 12;
    o.Password.RequireNonAlphanumeric = false; // priorizar largo sobre símbolos
    o.Lockout.MaxFailedAccessAttempts = 5;
    o.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    o.User.RequireUniqueEmail = true;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddSignInManager();

builder.Services.AddAuthentication(IdentityConstants.ApplicationScheme)
    .AddIdentityCookies();

builder.Services.ConfigureApplicationCookie(o =>
{
    o.Cookie.HttpOnly = true;
    o.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    o.Cookie.SameSite = SameSiteMode.Lax;
    o.ExpireTimeSpan = TimeSpan.FromHours(8);
});

var app = builder.Build();
app.UseHsts();
app.UseHttpsRedirection();
app.UseRateLimiter();   // con una política estricta para /auth/login (ver rate limiting)
app.UseAuthentication();
app.UseAuthorization();
```

`AuthController.cs`:

```csharp
[AllowAnonymous]
[ApiController]
[Route("api/v1/auth")]
public class AuthController(SignInManager<Usuario> signIn, ILogger<AuthController> logger) : ControllerBase
{
    [EnableRateLimiting("login")]
    [HttpPost("login")]
    public async Task<IActionResult> Login(LoginRequest req)
    {
        var resultado = await signIn.PasswordSignInAsync(
            req.Email, req.Password, isPersistent: false, lockoutOnFailure: true);

        if (resultado.Succeeded)
        {
            logger.LogInformation("Login exitoso de {Email} desde {Ip}", req.Email, HttpContext.Connection.RemoteIpAddress);
            return NoContent(); // la cookie HttpOnly ya quedó en la respuesta
        }

        logger.LogWarning("Login fallido de {Email} desde {Ip}. Bloqueado: {Bloqueado}",
            req.Email, HttpContext.Connection.RemoteIpAddress, resultado.IsLockedOut);

        // mismo mensaje para email inexistente, contraseña incorrecta o cuenta bloqueada
        return Problem(title: "Email o contraseña incorrectos", statusCode: StatusCodes.Status401Unauthorized);
    }
}

public record LoginRequest([property: Required, EmailAddress] string Email, [property: Required] string Password);
```

Qué observar:

* Las contraseñas se guardan con PBKDF2 y salt (Identity), y los hashes viejos se migran solos con `SuccessRehashNeeded`.
* Cinco intentos fallidos bloquean la cuenta 15 minutos, y el rate limiting frena los ataques desde una IP contra muchas cuentas.
* Un solo mensaje de error: no se puede saber qué emails están registrados.
* La sesión vive en una cookie `HttpOnly`, `Secure` y `SameSite`: un XSS no la lee y no viaja por HTTP.
* Los logs registran el email, la IP y el resultado, nunca la contraseña.
* HSTS y la redirección obligan a HTTPS.

-----

## Errores comunes

**1. Guardar contraseñas con MD5, SHA-1 o SHA-256.**
Qué pasa: si se filtra la base, la mayoría de las contraseñas se recuperan en horas.
Por qué: son hashes rápidos, pensados para integridad, no para contraseñas.
Arreglo: Argon2id, bcrypt o PBKDF2 con muchas iteraciones; en .NET, el `PasswordHasher` de Identity.

**2. Cifrar contraseñas.**
Qué pasa: quien obtenga la clave recupera todas las contraseñas.
Por qué: se confunde cifrado con hash.
Arreglo: hash con KDF; nunca necesitas leer una contraseña.

**3. Mensajes de error distintos en el login.**
Qué pasa: se puede saber qué emails están registrados.
Por qué: se busca ser "útil" con el usuario.
Arreglo: "Email o contraseña incorrectos" en todos los casos.

**4. Tokens largos en `localStorage`.**
Qué pasa: un XSS roba un token que sirve durante días y no se puede revocar.
Por qué: es la forma más simple de guardar el token en el frontend.
Arreglo: cookies `HttpOnly` (o un BFF) para aplicaciones web; tokens de acceso de vida corta con refresh tokens rotados.

**5. Datos sensibles en el JWT.**
Qué pasa: cualquiera que vea el token lee el documento, el teléfono o el saldo.
Por qué: se asume que el token está cifrado.
Arreglo: en el JWT, solo identificadores y roles; lo demás se consulta en el servidor.

**6. Contraseñas y tokens en los logs.**
Qué pasa: cualquiera con acceso a los logs obtiene credenciales.
Por qué: se registra el body completo de las peticiones.
Arreglo: logging estructurado de campos concretos; nunca el body de autenticación.

-----

## Evolución y versiones

* **1999:** bcrypt.
* **2015:** Argon2 gana la Password Hashing Competition.
* **2015:** se publica el estándar JWT (RFC 7519).
* **2017:** NIST SP 800-63B recomienda largo y listas de contraseñas filtradas en lugar de reglas de complejidad y cambios periódicos.
* **.NET 7 (2022):** el `PasswordHasher` de Identity pasa de HMAC-SHA256 con 10 000 iteraciones a HMAC-SHA512 con 100 000.
* **2017-2025:** "Broken Authentication" pasa a llamarse "Identification and Authentication Failures" (2021) y "Authentication Failures" (2025) en el OWASP Top 10.

-----

## Cuándo sí y cuándo no

**Usa cookies de sesión cuando:** el frontend y la API están en el mismo dominio (o detrás de un BFF). Es lo más simple y lo más seguro para aplicaciones web.

**Usa JWT cuando:** la API la consumen apps móviles, otros servicios o varios dominios, o necesitas que los servicios validen la identidad sin consultar un servidor central.

**No implementes tu propio sistema de identidad cuando:** un proveedor estándar (Identity, Keycloak, Auth0, Entra ID) cubre tus necesidades. Es de los lugares donde inventar tiene más riesgo.

-----

## Resumen en 5 líneas

1. Las contraseñas se guardan con un hash lento y con salt (Argon2id, bcrypt, PBKDF2), nunca cifradas ni con SHA-256.
2. Contra la fuerza bruta: bloqueo por intentos, rate limiting, MFA y mensajes de error genéricos.
3. Cookies `HttpOnly`, `Secure` y `SameSite`; un JWT está firmado, no cifrado, y se valida completo con vida corta.
4. HTTPS siempre (con HSTS); cifrado en reposo solo para lo que hay que leer; y no guardar lo que no hace falta.
5. Nunca contraseñas, tokens ni datos sensibles en logs, en respuestas ni en el payload de un JWT.

-----

## Para profundizar

<details>
<summary>El patrón BFF (Backend for Frontend)</summary>

En una SPA o en Next.js, el patrón BFF pone un pequeño backend propio entre el navegador y las APIs. El BFF hace el login con el proveedor de identidad, **guarda los tokens del lado del servidor** y le da al navegador solo una cookie `HttpOnly` de sesión. Cuando el frontend llama al BFF, este agrega el token de acceso y reenvía la petición a la API.

Resultado: los tokens nunca llegan al JavaScript del navegador, así que un XSS no puede robarlos. Es la recomendación actual del IETF para aplicaciones de navegador que usan OAuth.

</details>

<details>
<summary>Passkeys: autenticación sin contraseñas</summary>

Las **passkeys** (basadas en WebAuthn/FIDO2) reemplazan la contraseña por un par de claves: la privada queda en el dispositivo del usuario (protegida por huella, rostro o PIN) y el servidor solo guarda la pública. No hay nada que robar de la base de datos, y no se pueden usar en un sitio de phishing, porque están ligadas al dominio. .NET 10 agrega soporte para passkeys en ASP.NET Core Identity.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Las contraseñas se guardan con hash y salt, usando algoritmos como bcrypt o el `PasswordHasher` de Identity. Hay que usar HTTPS, limitar los intentos de login y no guardar datos sensibles en los logs. Un JWT tiene los datos del usuario firmados.

### Respuesta ampliada (semi-senior)

Para contraseñas uso un hash lento con salt: en .NET, el `PasswordHasher` de Identity con PBKDF2, o Argon2id si lo implemento aparte; nunca SHA-256 ni cifrado. Contra la fuerza bruta combino bloqueo por cuenta, rate limiting por IP, MFA y mensajes de error genéricos para no permitir la enumeración. Para aplicaciones web prefiero cookies `HttpOnly`, `Secure` y `SameSite`, y si uso JWT, valido firma, emisor, audiencia y expiración, fijo el algoritmo, uso tokens de vida corta con refresh tokens rotados, y no los guardo en `localStorage`. Los datos sensibles van por HTTPS con HSTS, cifrados en reposo con Data Protection si hay que leerlos, y nunca en logs ni en el payload del token. Y siempre que puedo, uso un proveedor de identidad estándar en lugar de construirlo yo.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué no usar SHA-256 para contraseñas?**
Porque es muy rápido: una GPU prueba miles de millones por segundo. Para contraseñas hace falta una función lenta a propósito y con salt.

**2. ¿Un JWT es seguro para guardar datos sensibles?**
No. Está firmado, no cifrado: cualquiera puede leer el payload. La firma solo garantiza que no fue modificado.

**3. ¿Dónde guardarías el token en una SPA?**
Idealmente en ningún lugar accesible por JavaScript: una cookie `HttpOnly` o el patrón BFF. En `localStorage`, cualquier XSS lo roba.

**4. ¿Qué es el salt y para qué sirve?**
Un valor aleatorio por usuario que se agrega antes de calcular el hash. Hace que la misma contraseña dé hashes distintos e inutiliza las tablas precalculadas.

-----

## Práctica

**Ejercicio 1.** Encuentra los problemas de este endpoint de recuperación de contraseña.

```csharp
[HttpPost("forgot-password")]
public async Task<IActionResult> Olvide(string email)
{
    var u = await db.Users.FirstOrDefaultAsync(x => x.Email == email);
    if (u is null) return NotFound("No existe una cuenta con ese email");

    u.ResetToken = new Random().Next(1000, 9999).ToString();
    await db.SaveChangesAsync();
    logger.LogInformation("Token de reseteo {Token} para {Email}", u.ResetToken, email);
    await emailService.EnviarAsync(u.Email, $"Tu código es {u.ResetToken}");
    return Ok();
}
```

<details>
<summary>Solución</summary>

* **Enumeración:** `404` revela qué emails están registrados. Debe responder siempre lo mismo.
* **Token débil:** 4 dígitos son 9000 combinaciones, y `Random` no es criptográficamente seguro. Usa `RandomNumberGenerator.GetBytes(32)` o el `GeneratePasswordResetTokenAsync` de Identity.
* **Sin expiración:** el token sirve para siempre.
* **Token en texto plano en la base y en el log:** quien vea la base o los logs puede restablecer contraseñas ajenas. Guarda solo su hash y nunca lo registres.
* **Sin rate limiting:** se pueden pedir y probar códigos sin límite.

</details>

**Ejercicio 2.** Decodifica mentalmente este payload de un JWT. ¿Qué problema tiene?

```json
{ "sub": "42", "email": "ana@mail.com", "dni": "45781236", "tarjeta": "4111111111111111", "role": "Cliente", "exp": 1924992000 }
```

<details>
<summary>Solución</summary>

* **Datos sensibles en el payload:** el DNI y el número de tarjeta los puede leer cualquiera que vea el token (logs, extensiones del navegador, herramientas de depuración). En el token van solo identificadores y roles.
* **Expiración en 2031:** un token de acceso tiene que durar minutos, no años. Si lo roban, no se puede revocar.

</details>

-----

## Siguiente lección

[Gestión de secretos y configuración por ambiente](06-gestion-de-secretos.md)
