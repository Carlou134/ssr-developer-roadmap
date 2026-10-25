# Gestión de secretos y configuración por ambiente

## En una frase

Un secreto (contraseña de base de datos, API key, clave de firma) **nunca vive en el código ni en el repositorio**: en desarrollo va en User Secrets, en producción en un gestor de secretos o en variables de entorno inyectadas por la plataforma, cada ambiente tiene los suyos, y si uno se filtra **se rota**, porque borrarlo del repositorio no alcanza.

-----

## Antes de empezar

Conviene que ya sepas:

* El patrón Options y la validación de configuración: [Inyección de dependencias](../02-calidad-de-codigo-y-arquitectura/06-inyeccion-de-dependencias.md).
* Cómo se configura `appsettings.json`: [Interpretación de logs](../01-autonomia-resolucion-problemas/03-interpretacion-de-logs.md).
* Git básico: commits e historial.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Secreto:** dato que da acceso a algo: contraseña, cadena de conexión, API key, clave privada, token.
* **Configuración:** valores que cambian entre ambientes pero no son secretos: URLs, timeouts, niveles de log.
* **Ambiente (*environment*):** cada lugar donde corre la aplicación: desarrollo, pruebas, *staging*, producción.
* **User Secrets:** almacén de secretos de .NET para desarrollo, guardado fuera del repositorio, en el perfil del usuario.
* **Variable de entorno:** valor que el sistema operativo o la plataforma le pasa al proceso al iniciarlo.
* **Gestor de secretos (*secrets manager*):** servicio que guarda, cifra, controla el acceso y audita los secretos: Azure Key Vault, AWS Secrets Manager, HashiCorp Vault.
* **Identidad administrada (*managed identity*):** identidad que la nube asigna a tu aplicación, para que acceda a otros servicios sin una contraseña guardada.
* **Rotación:** reemplazar un secreto por uno nuevo y revocar el anterior.
* **Escaneo de secretos:** herramienta que busca secretos en el código y en el historial de Git.

-----

## El problema

El `appsettings.json` del proyecto, en el repositorio:

```json
{
  "ConnectionStrings": {
    "Pedidos": "Server=prod-sql.empresa.com;Database=Pedidos;User Id=sa;Password=Empresa2024!;"
  },
  "Stripe": { "SecretKey": "sk_live_51H8xK..." },
  "Jwt": { "SigningKey": "mi-clave-super-secreta" }
}
```

Un día alguien hace público el repositorio para mostrarlo en su portafolio. Lo vuelven privado a las dos horas y borran el archivo en un nuevo commit.

Lo que pasa después:

* bots que escanean GitHub de forma continua encuentran la clave de Stripe en minutos y empiezan a hacer reembolsos fraudulentos;
* el archivo sigue en el **historial** de Git: cualquier clon hecho en esas dos horas lo tiene;
* la cadena de conexión usa `sa`, el administrador de SQL Server, para producción: quien la tenga controla todo el servidor;
* con la clave de firma del JWT, cualquiera puede generar tokens válidos con el rol que quiera;
* desarrollo, pruebas y producción usan las mismas credenciales: un error en las pruebas afecta a los datos reales.

-----

## Cómo funciona

### 1. Secreto frente a configuración

| | Configuración | Secreto |
| --- | --- | --- |
| Ejemplos | URL de una API, timeout, nivel de log, `pageSize` máximo | Contraseñas, cadenas de conexión, API keys, claves de firma, certificados |
| ¿Puede estar en el repositorio? | Sí (`appsettings.json`) | **Nunca** |
| ¿Cambia por ambiente? | A menudo | Siempre: cada ambiente tiene los suyos |
| Si se filtra | No pasa nada grave | Hay que rotarlo de inmediato |

Una prueba rápida: **si este valor apareciera en un tweet, ¿tendría que cambiarlo?** Si la respuesta es sí, es un secreto.

### 2. Cómo arma .NET la configuración

ASP.NET Core lee la configuración de varias fuentes, en orden. **La última que define una clave gana:**

```text
1. appsettings.json                    ← valores por defecto, en el repositorio
2. appsettings.{Environment}.json      ← ajustes por ambiente, en el repositorio
3. User Secrets                        ← solo en Development, fuera del repositorio
4. Variables de entorno                ← la plataforma las inyecta
5. Argumentos de línea de comandos
```

El ambiente se define con la variable `ASPNETCORE_ENVIRONMENT` (`Development`, `Staging`, `Production`). Si no está definida, es `Production`.

Esto permite un esquema limpio: `appsettings.json` tiene la **forma** de la configuración y los valores no sensibles; los secretos llegan desde una fuente posterior que los sobrescribe.

```json
{
  "ConnectionStrings": { "Pedidos": "" },
  "Stripe": { "SecretKey": "" }
}
```

### 3. Desarrollo: User Secrets

User Secrets guarda los secretos de desarrollo en un archivo JSON **en tu perfil de usuario**, fuera de la carpeta del proyecto, así que nunca termina en un commit.

```text
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Pedidos" "Server=localhost;Database=Pedidos;Trusted_Connection=True;TrustServerCertificate=True"
dotnet user-secrets set "Stripe:SecretKey" "sk_test_51H8xK..."
dotnet user-secrets list
```

* `init` agrega un `UserSecretsId` al `.csproj`.
* Los valores se guardan en `%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` (Windows) o `~/.microsoft/usersecrets/<id>/secrets.json` (Linux y macOS).
* `WebApplication.CreateBuilder` los carga solo cuando el ambiente es `Development`.

> **Ojo:** User Secrets **no cifra** nada: es un archivo de texto fuera del repositorio. Sirve para no commitear secretos, no para producción.

### 4. Producción: variables de entorno y gestores de secretos

**Variables de entorno.** La plataforma (Azure App Service, Kubernetes, Docker, un pipeline) las inyecta al iniciar el proceso. En .NET, la jerarquía se separa con **doble guion bajo**:

```text
ConnectionStrings__Pedidos=Server=prod-sql...;User Id=app_pedidos;Password=...
Stripe__SecretKey=sk_live_...
```

`ConnectionStrings__Pedidos` es la clave `ConnectionStrings:Pedidos`. (El `:` no funciona en variables de entorno en todos los sistemas operativos).

**Gestores de secretos.** Para producción, lo recomendado es un servicio dedicado:

| Ventaja | Por qué importa |
| --- | --- |
| Cifrado y control de acceso | Solo la aplicación y las personas autorizadas pueden leer cada secreto |
| Auditoría | Registra quién leyó o modificó cada secreto y cuándo |
| Rotación | Permite cambiar un secreto sin redesplegar, e incluso rotarlo automáticamente |
| Versionado | Si una rotación falla, se puede volver a la versión anterior |

Con **Azure Key Vault** e **identidad administrada**, la aplicación lee los secretos sin tener **ninguna** credencial guardada:

```csharp
// paquetes: Azure.Extensions.AspNetCore.Configuration.Secrets y Azure.Identity
if (builder.Environment.IsProduction())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri("https://kv-tienda-prod.vault.azure.net/"),
        new DefaultAzureCredential());
}
```

En Key Vault, los nombres usan `--` como separador: el secreto `ConnectionStrings--Pedidos` se convierte en la clave `ConnectionStrings:Pedidos`. `DefaultAzureCredential` usa la identidad administrada en Azure y tu sesión de Azure CLI o Visual Studio en tu máquina.

Las alternativas en otras nubes funcionan igual: AWS Secrets Manager con roles de IAM, Google Secret Manager con cuentas de servicio, o HashiCorp Vault fuera de la nube.

### 5. Separación por ambiente y mínimo privilegio

Cada ambiente tiene **sus propios** secretos, y cada secreto tiene **los mínimos permisos** necesarios:

| | Desarrollo | Staging | Producción |
| --- | --- | --- | --- |
| Base de datos | Local o de desarrollo | Copia sin datos reales | Real |
| Usuario de base | El tuyo | `app_pedidos_stg` | `app_pedidos` (lectura y escritura, sin permisos de esquema) |
| Pagos | Claves de prueba (`sk_test_`) | Claves de prueba | Claves reales (`sk_live_`) |
| Dónde viven los secretos | User Secrets | Key Vault de staging | Key Vault de producción |
| Quién los puede leer | Tú | El equipo | Solo la aplicación y dos personas de guardia |

Así, una filtración de los secretos de desarrollo no compromete producción, y un error en las pruebas no toca datos reales.

### 6. El frontend no tiene secretos

Todo lo que se envía al navegador **lo puede leer cualquiera**. Las variables de entorno con prefijo público se incrustan en el JavaScript al compilar:

| Framework | Prefijo público | Qué significa |
| --- | --- | --- |
| Next.js | `NEXT_PUBLIC_` | Se copia en el bundle del cliente |
| Vite | `VITE_` | Se copia en el bundle del cliente |

`NEXT_PUBLIC_API_URL` está bien: es configuración. `NEXT_PUBLIC_STRIPE_SECRET_KEY` es un secreto publicado. Las claves secretas solo se usan en el servidor (Server Components, Route Handlers, el backend).

> **Ojo:** algunas claves están **diseñadas** para ser públicas, como la clave publicable de Stripe (`pk_live_...`) o la clave de un mapa restringida por dominio. La regla es leer la documentación del proveedor: si dice "secreta", nunca va al frontend.

### 7. Si un secreto se filtra: rotar, no solo borrar

Borrar el secreto en un commit nuevo **no lo elimina**: sigue en el historial de Git, en cada clon, en cada fork y, si el repositorio fue público, posiblemente en los registros de los bots que escanean GitHub.

El orden correcto:

```text
1. ROTAR      → generar un secreto nuevo en el proveedor y revocar el filtrado (minutos, no días)
2. DESPLEGAR  → actualizar la aplicación con el secreto nuevo
3. INVESTIGAR → revisar los logs del proveedor: ¿alguien usó el secreto filtrado?
4. LIMPIAR    → reescribir el historial (git filter-repo o BFG) solo si hace falta, sabiendo que no recupera los clones existentes
5. PREVENIR   → escaneo de secretos en el pipeline y en cada push
```

El paso 1 es el único que de verdad protege. Los demás son limpieza y prevención.

### 8. Prevenir: que el secreto nunca llegue al repositorio

| Capa | Herramienta |
| --- | --- |
| `.gitignore` | `.env`, `.env.local`, `appsettings.*.local.json`, `*.pfx` |
| Antes del commit | Hook de pre-commit con gitleaks |
| Al hacer push | Protección de push de GitHub (*secret scanning push protection*): bloquea el push si detecta un secreto conocido |
| En el pipeline | gitleaks sobre el historial completo en cada PR |
| Después | Escaneo de secretos del repositorio, con alertas |

```text
# .gitignore
.env
.env.*
!.env.example
appsettings.*.local.json
*.pfx
```

Un `.env.example` con las claves **vacías** documenta qué variables necesita el proyecto sin exponer valores.

-----

## Ejemplo completo

La configuración del problema, corregida.

`appsettings.json` (en el repositorio, sin secretos):

```json
{
  "ConnectionStrings": { "Pedidos": "" },
  "Stripe": { "SecretKey": "", "WebhookUrl": "https://tienda.com/webhooks/stripe" },
  "Jwt": { "Issuer": "https://auth.tienda.com", "Audience": "api-tienda", "SigningKey": "" }
}
```

`StripeOptions.cs` (los secretos también se validan al arrancar):

```csharp
public class StripeOptions
{
    [Required] public string SecretKey { get; set; } = string.Empty;
    [Required, Url] public string WebhookUrl { get; set; } = string.Empty;
}
```

`Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args); // carga appsettings, User Secrets (Development) y variables de entorno

if (builder.Environment.IsProduction())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri(builder.Configuration["KeyVault:Url"]!),
        new DefaultAzureCredential());
}

builder.Services.AddOptions<StripeOptions>()
    .Bind(builder.Configuration.GetSection("Stripe"))
    .ValidateDataAnnotations()
    .ValidateOnStart(); // si falta el secreto, la aplicación no arranca
```

Desarrollo:

```text
dotnet user-secrets set "ConnectionStrings:Pedidos" "Server=localhost;Database=Pedidos_Dev;Trusted_Connection=True;TrustServerCertificate=True"
dotnet user-secrets set "Stripe:SecretKey" "sk_test_..."
dotnet user-secrets set "Jwt:SigningKey" "una-clave-local-de-al-menos-32-caracteres"
```

Producción (Azure App Service):

```text
App Settings:   KeyVault__Url = https://kv-tienda-prod.vault.azure.net/
Key Vault:      ConnectionStrings--Pedidos   (usuario app_pedidos, sin permisos de esquema)
                Stripe--SecretKey            (sk_live_...)
                Jwt--SigningKey              (generada con 256 bits aleatorios)
Acceso:         la identidad administrada de la App Service tiene solo permiso de lectura de secretos
```

Pipeline:

```yaml
- name: Buscar secretos en el historial
  uses: gitleaks/gitleaks-action@v2
```

Qué observar:

* El repositorio puede ser público sin exponer nada: solo tiene la forma de la configuración.
* La aplicación en producción no tiene **ninguna** contraseña guardada: entra a Key Vault con su identidad administrada.
* Producción ya no usa `sa`: el usuario de la aplicación tiene los mínimos permisos.
* Si falta un secreto, la aplicación falla al arrancar, no al primer cobro.
* Los secretos filtrados del problema se rotaron **antes** de limpiar el historial.

-----

## Errores comunes

**1. Secretos en `appsettings.json` o en el código.**
Qué pasa: cualquiera con acceso al repositorio, hoy o en el futuro, los tiene.
Por qué: es lo más rápido para "que funcione".
Arreglo: User Secrets en desarrollo; variables de entorno o un gestor de secretos en producción.

**2. Borrar el secreto en lugar de rotarlo.**
Qué pasa: el secreto sigue válido y sigue en el historial y en los clones.
Por qué: se cree que eliminar el archivo lo elimina.
Arreglo: rotar primero, siempre; limpiar el historial es secundario.

**3. Las mismas credenciales en todos los ambientes.**
Qué pasa: un error en pruebas afecta producción, y una filtración de desarrollo compromete todo.
Por qué: "es más simple tener una sola cadena de conexión".
Arreglo: secretos distintos por ambiente, con permisos mínimos.

**4. Secretos en variables públicas del frontend.**
Qué pasa: la clave aparece en el JavaScript que descarga cualquier visitante.
Por qué: se usa `NEXT_PUBLIC_` o `VITE_` para que "funcione en el cliente".
Arreglo: las claves secretas solo en el servidor; el frontend llama a tu backend.

**5. Credenciales de administrador para la aplicación.**
Qué pasa: una inyección SQL o una filtración da control total del servidor de base de datos.
Por qué: `sa` o `root` "siempre funcionan".
Arreglo: un usuario por aplicación, con solo los permisos que necesita.

**6. Secretos en los logs o en mensajes de error.**
Qué pasa: la cadena de conexión aparece en un stack trace o en el log de arranque.
Por qué: se registra la configuración completa para depurar.
Arreglo: nunca registres la configuración cruda; si hace falta, registra solo si un valor está presente, no el valor.

-----

## Evolución y versiones

* **ASP.NET Core 1.0 (2016):** configuración por capas, variables de entorno y User Secrets.
* **2018:** el proveedor de configuración de Azure Key Vault para ASP.NET Core.
* **2020:** `DefaultAzureCredential` en el SDK de Azure unifica la autenticación local y en la nube.
* **2023:** la protección de push de secretos de GitHub queda disponible para todos los repositorios públicos.
* **.NET 6:** `ValidateOnStart()` permite que una configuración inválida o un secreto faltante impidan arrancar.

-----

## Cuándo sí y cuándo no

**Usa User Secrets cuando:** desarrollas en tu máquina.

**Usa variables de entorno cuando:** la plataforma las gestiona de forma segura (los secretos de Kubernetes, la configuración de Azure App Service, los secretos de GitHub Actions) y no necesitas rotación ni auditoría avanzadas.

**Usa un gestor de secretos cuando:** es producción, hay datos o dinero reales, varios servicios comparten secretos, o necesitas auditoría y rotación.

**No pongas secretos en:** el código, el repositorio, las imágenes de Docker (`ENV` en un Dockerfile queda en la imagen), los logs ni el frontend.

-----

## Resumen en 5 líneas

1. Un secreto da acceso a algo; si aparecer en público te obligaría a cambiarlo, no va en el repositorio.
2. .NET combina fuentes en orden: `appsettings` → User Secrets (desarrollo) → variables de entorno → línea de comandos; gana la última.
3. Producción: gestor de secretos con identidad administrada, para no tener ninguna credencial guardada.
4. Secretos distintos por ambiente y con mínimos permisos; el frontend nunca tiene secretos.
5. Si se filtra, se rota primero; y se previene con `.gitignore`, gitleaks y la protección de push.

-----

## Para profundizar

<details>
<summary>Secretos de Kubernetes: Base64 no es cifrado</summary>

Un `Secret` de Kubernetes guarda los valores en **Base64**, que es solo una codificación: `echo "c2VjcmV0bw==" | base64 -d` devuelve `secreto`. Por defecto, además, se almacenan sin cifrar en la base de datos del clúster (etcd). Para protegerlos de verdad: cifrado en reposo de etcd, control de acceso (RBAC) estricto sobre quién puede leer secretos, y preferiblemente un gestor externo sincronizado con herramientas como External Secrets Operator o el CSI driver de secretos.

</details>

<details>
<summary>Rotación automática y secretos dinámicos</summary>

El mejor secreto es el que dura poco. Los gestores modernos pueden:

* **rotar automáticamente** un secreto cada cierto tiempo, actualizando el valor en el servicio de origen (por ejemplo, la contraseña de una base de datos);
* generar **secretos dinámicos** (HashiCorp Vault): cada vez que la aplicación arranca, recibe un usuario y una contraseña de base de datos nuevos, que expiran en horas.

Si un secreto dinámico se filtra, deja de servir solo. Para leer siempre el valor actual, la aplicación tiene que recargar la configuración (por ejemplo, con `IOptionsMonitor` y el intervalo de recarga del proveedor de Key Vault).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Los secretos, como contraseñas y API keys, no se guardan en el código ni en el repositorio. En desarrollo uso User Secrets de .NET y en producción variables de entorno o un servicio como Azure Key Vault.

### Respuesta ampliada (semi-senior)

Separo configuración de secretos: el `appsettings.json` del repositorio solo tiene la forma y los valores no sensibles. En desarrollo uso User Secrets; en producción, un gestor de secretos como Key Vault, al que la aplicación accede con identidad administrada, así no hay ninguna credencial guardada. Cada ambiente tiene sus propios secretos con permisos mínimos, y valido al arrancar que estén todos con `ValidateOnStart`. En el frontend no hay secretos: todo lo que tiene prefijo público queda en el bundle. Lo prevengo con `.gitignore`, gitleaks en el pipeline y la protección de push de GitHub, y si algo se filtra, lo primero es rotarlo, porque borrarlo del repositorio no lo invalida.

### Preguntas frecuentes de seguimiento

**1. Commiteaste una API key por error. ¿Qué haces?**
La roto de inmediato en el proveedor, despliego con la nueva, reviso en los logs del proveedor si se usó, y después evalúo limpiar el historial. Borrar el archivo no alcanza.

**2. ¿User Secrets sirve para producción?**
No. Es un archivo de texto sin cifrar en el perfil del usuario, pensado para no commitear secretos en desarrollo.

**3. ¿Qué ventaja tiene una identidad administrada?**
Que la aplicación accede a otros servicios sin ninguna contraseña guardada: la nube gestiona y rota sus credenciales.

-----

## Práctica

**Ejercicio 1.** Clasifica cada valor como configuración o secreto, y di dónde lo pondrías en producción.

1. La URL de la API de pagos.
2. La clave secreta de la API de pagos.
3. El timeout de las llamadas HTTP.
4. La cadena de conexión a la base de datos.
5. La clave publicable de Stripe para el frontend.
6. La clave con la que se firman los JWT.

<details>
<summary>Solución</summary>

1. **Configuración:** `appsettings.json`.
2. **Secreto:** gestor de secretos.
3. **Configuración:** `appsettings.json` o `appsettings.Production.json`.
4. **Secreto:** gestor de secretos (o, mejor, identidad administrada contra la base de datos, sin contraseña).
5. **Configuración pública:** está diseñada para ir al navegador; puede ser `NEXT_PUBLIC_`.
6. **Secreto:** gestor de secretos. Quien la tenga puede fabricar tokens con cualquier rol.

</details>

**Ejercicio 2.** ¿Qué problemas tiene este Dockerfile? Corrígelo.

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0
COPY ./publish /app
ENV ConnectionStrings__Pedidos="Server=prod;User Id=sa;Password=Empresa2024!"
ENV ASPNETCORE_ENVIRONMENT=Production
ENTRYPOINT ["dotnet", "/app/Tienda.dll"]
```

<details>
<summary>Solución</summary>

* **El secreto queda dentro de la imagen:** cualquiera que pueda descargar la imagen lo ve con `docker history` o `docker inspect`.
* **Usa `sa`:** el administrador del servidor para una aplicación.
* **Producción y la imagen quedan atadas:** para cambiar la contraseña hay que reconstruir la imagen.

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0
COPY ./publish /app
ENV ASPNETCORE_ENVIRONMENT=Production
ENTRYPOINT ["dotnet", "/app/Tienda.dll"]
```

La cadena de conexión, con un usuario de mínimos permisos, la inyecta la plataforma al ejecutar el contenedor (variables de entorno del orquestador o, mejor, un gestor de secretos). La misma imagen sirve para todos los ambientes.

</details>

-----

## Siguiente lección

[Resiliencia: retries, timeouts y circuit breaker](07-resiliencia-con-polly.md)
