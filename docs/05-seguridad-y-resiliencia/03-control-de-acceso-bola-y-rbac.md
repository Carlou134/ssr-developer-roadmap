# Control de acceso: BOLA y RBAC

## En una frase

Autenticar responde "¿quién eres?"; autorizar responde "¿puedes hacer **esto** con **este** recurso?". El riesgo número uno en APIs, **BOLA**, aparece cuando se verifica lo primero y se olvida lo segundo; se previene comprobando en el servidor, en cada petición, que el recurso pertenece al usuario, y combinándolo con **roles (RBAC)** y políticas para las acciones.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es el Top 10 y por qué el control de acceso es el primer riesgo: [OWASP Top 10](02-owasp-top-10.md).
* Diferencia entre `401` y `403`: [Estándares RESTful](../03-diseno-de-apis-y-contratos/01-estandares-restful.md).
* Qué es un DTO: [Arquitectura en capas](../02-calidad-de-codigo-y-arquitectura/08-arquitectura-en-capas-ntier.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Autenticación (AuthN):** verificar la identidad: ¿quién eres?
* **Autorización (AuthZ):** verificar los permisos: ¿puedes hacer esto?
* **BOLA (*Broken Object Level Authorization*):** acceder a un objeto ajeno cambiando su identificador. También se llama **IDOR** (*Insecure Direct Object Reference*).
* **BFLA (*Broken Function Level Authorization*):** ejecutar una función (un endpoint) que no corresponde a tu rol.
* **Asignación masiva (*mass assignment*):** el servidor copia al objeto todos los campos que envía el cliente, incluidos los que no debería poder modificar.
* **RBAC (*Role-Based Access Control*):** permisos asignados según el rol del usuario: administrador, vendedor, cliente.
* **ABAC (*Attribute-Based Access Control*):** permisos según atributos del usuario, del recurso o del contexto: dueño, departamento, horario.
* **Claim:** un dato sobre el usuario incluido en su identidad, como su Id, su email o sus roles.
* **Política de autorización:** regla con nombre que define qué se exige para acceder, como "solo administradores".
* **Autorización basada en recursos:** decidir el permiso según el recurso concreto, no solo según el usuario.

-----

## El problema

La API de una tienda tiene autenticación con JWT. Todos los endpoints tienen `[Authorize]`.

```csharp
[Authorize]
[ApiController]
[Route("api/v1/orders")]
public class OrdersController(AppDbContext db) : ControllerBase
{
    [HttpGet("{id:int}")]
    public async Task<ActionResult<Order>> Get(int id) =>
        await db.Orders.FindAsync(id) is { } o ? o : NotFound();

    [HttpPut("{id:int}")]
    public async Task<IActionResult> Update(int id, Order cambios)
    {
        var order = await db.Orders.FindAsync(id);
        if (order is null) return NotFound();
        db.Entry(order).CurrentValues.SetValues(cambios);
        await db.SaveChangesAsync();
        return NoContent();
    }

    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id) { /* borra cualquier pedido */ }
}
```

Un cliente, autenticado correctamente, descubre que:

* con `GET /api/v1/orders/1044`, `1045`, `1046`... descarga los pedidos de **todos** los clientes, con nombre y dirección (**BOLA**);
* con `PUT /api/v1/orders/1043` y `"total": 0.01, "estado": "Pagado"` en el body, cambia el total y el estado de su propio pedido (**asignación masiva**);
* `DELETE /api/v1/orders/{id}`, que en el frontend solo ve el administrador, funciona para cualquier usuario (**BFLA**).

Las consecuencias: `[Authorize]` solo verificó que hubiera **alguien** autenticado. Nadie verificó **qué** podía hacer ni **con qué** recurso.

-----

## Cómo funciona

### 1. Autenticación no es autorización

| | Autenticación | Autorización |
| --- | --- | --- |
| Pregunta | ¿Quién eres? | ¿Puedes hacer esto con este recurso? |
| Si falla | `401 Unauthorized` | `403 Forbidden` (o `404`, ver abajo) |
| En ASP.NET Core | `UseAuthentication()`, JWT, cookies | `UseAuthorization()`, `[Authorize]`, políticas, verificaciones en el código |
| Ejemplo | El token es válido y pertenece a Ana | Ana puede ver el pedido 1043 porque es suyo |

`[Authorize]` sin parámetros solo responde la primera pregunta. **La autorización a nivel de objeto la tienes que escribir tú**, en cada endpoint que recibe un identificador.

### 2. BOLA: el riesgo número uno en APIs

BOLA ocurre cuando la API usa el Id que llega del cliente para buscar un recurso, sin verificar que ese recurso le pertenece al usuario. Es el primer riesgo del OWASP API Security Top 10.

La URL también es entrada del usuario: que el frontend no muestre links a pedidos ajenos no impide que alguien escriba la URL a mano o recorra Ids con un script.

**La corrección:** el dueño del recurso se toma de la **identidad autenticada** (los claims del token o la cookie), nunca del cliente, y se usa como filtro en la consulta.

```csharp
[HttpGet("{id:int}")]
public async Task<ActionResult<OrderDto>> Get(int id, CancellationToken ct)
{
    var usuarioId = User.FindFirstValue(ClaimTypes.NameIdentifier)!;

    var pedido = await db.Orders
        .Where(o => o.Id == id && o.CustomerId == usuarioId) // el filtro ES la autorización
        .Select(o => new OrderDto(o.Id, o.Total, o.Estado))
        .FirstOrDefaultAsync(ct);

    return pedido is null ? NotFound() : pedido;
}
```

Si lo usas mal:

```csharp
[HttpGet]
public Task<List<Order>> Mine([FromQuery] string customerId) =>
    db.Orders.Where(o => o.CustomerId == customerId).ToListAsync();
```

`GET /api/v1/orders?customerId=otro-usuario` devuelve los pedidos de otro. Aunque el parámetro se llame "mis pedidos", el Id del dueño viene del cliente, que puede poner lo que quiera.

### 3. ¿`403` o `404` cuando el recurso es de otro?

| Respuesta | Qué le dice al atacante |
| --- | --- |
| `403 Forbidden` | "El pedido 1044 **existe**, pero no es tuyo" |
| `404 Not Found` | "No hay ningún pedido 1044 para ti" |

Responder `403` permite **enumerar** qué Ids existen. Para recursos que pertenecen a un usuario, lo habitual es devolver `404`, igual que si no existiera. El `403` se reserva para cuando el usuario sabe que el recurso existe pero no tiene permiso sobre la acción (por ejemplo, un vendedor que intenta borrar un producto que puede ver).

### 4. ¿Un GUID en lugar de un número secuencial lo resuelve?

No. Un GUID (`3f2a9c1e-...`) hace más difícil **adivinar** Ids, y eso es una mejora razonable porque evita la enumeración y no revela cuántos pedidos tienes. Pero el GUID aparece en URLs compartidas, en logs, en el historial del navegador y en respuestas de otros endpoints. Si alguien lo obtiene y la API no verifica el dueño, accede igual. **Un Id difícil de adivinar es defensa en profundidad, no un control de acceso.**

### 5. Asignación masiva (BOPLA)

Si el endpoint recibe la **entidad** completa, el cliente puede enviar campos que no debería poder modificar:

```http
PUT /api/v1/orders/1043
{ "direccion": "Av. Lima 123", "total": 0.01, "estado": "Pagado", "customerId": "otro" }
```

`SetValues(cambios)` copia **todo**: el total, el estado y hasta el dueño. Lo mismo pasa al registrarse con `"isAdmin": true` en el body.

**La corrección:** un DTO de entrada **por operación**, con solo los campos que el usuario puede cambiar, y el mapeo explícito a la entidad:

```csharp
public record ActualizarDireccionRequest(string Direccion);

[HttpPut("{id:int}/direccion")]
public async Task<IActionResult> ActualizarDireccion(int id, ActualizarDireccionRequest req, CancellationToken ct)
{
    var usuarioId = User.FindFirstValue(ClaimTypes.NameIdentifier)!;
    var pedido = await db.Orders.FirstOrDefaultAsync(o => o.Id == id && o.CustomerId == usuarioId, ct);
    if (pedido is null) return NotFound();

    pedido.CambiarDireccion(req.Direccion); // regla de negocio en el dominio
    await db.SaveChangesAsync(ct);
    return NoContent();
}
```

Es la otra cara de los DTOs de [Arquitectura en capas](../02-calidad-de-codigo-y-arquitectura/08-arquitectura-en-capas-ntier.md): el DTO de salida evita **exponer** campos (`passwordHash`, `costoInterno`), y el de entrada evita **recibir** campos que no corresponden.

### 6. RBAC: permisos por rol

Para las **acciones** (qué endpoints puede llamar cada tipo de usuario), RBAC asigna permisos según el rol:

| Rol | Ver sus pedidos | Ver todos los pedidos | Cancelar pedidos | Borrar productos |
| --- | --- | --- | --- | --- |
| Cliente | ✅ | ❌ | Solo los suyos, si no se enviaron | ❌ |
| Vendedor | ✅ | ✅ | ✅ | ❌ |
| Administrador | ✅ | ✅ | ✅ | ✅ |

En ASP.NET Core, los roles llegan como claims (`ClaimTypes.Role`) en el token o la cookie:

```csharp
[Authorize(Roles = "Admin")]
[HttpDelete("{id:int}")]
public async Task<IActionResult> Delete(int id) { /* ... */ }
```

Mejor todavía: **políticas con nombre**, que separan la regla del endpoint y se pueden cambiar en un solo lugar:

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("GestionPedidos", p => p.RequireRole("Vendedor", "Admin"))
    .AddPolicy("SoloAdmin", p => p.RequireRole("Admin"))
    .AddPolicy("ClienteVerificado", p => p.RequireClaim("email_verified", "true"));

// en el controller
[Authorize(Policy = "GestionPedidos")]
[HttpGet("todos")]
public Task<List<OrderDto>> Todos() { /* ... */ }
```

`RequireRole("Vendedor", "Admin")` exige **cualquiera** de los dos roles. Varios `[Authorize]` sobre el mismo endpoint exigen **todos**.

> **Ojo con BFLA:** ocultar el botón "Borrar" en el frontend no es control de acceso. El endpoint existe y cualquiera puede llamarlo con Postman o `curl`. **Toda** regla de permisos se verifica en el servidor; el frontend solo la refleja para la experiencia de usuario.

### 7. Denegar por defecto

Con `[Authorize]` endpoint por endpoint, basta con olvidarlo en uno para dejarlo público. Es más seguro invertirlo: **todo requiere autenticación**, y lo público se marca explícitamente.

```csharp
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build());

// lo público, explícito
[AllowAnonymous]
[HttpGet("/api/v1/products")]
public Task<List<ProductDto>> Catalogo() { /* ... */ }
```

La *fallback policy* se aplica a todo endpoint que no tiene ningún atributo de autorización. Así, un endpoint nuevo nace protegido.

### 8. Autorización basada en recursos

A veces la regla depende **del recurso y del rol a la vez**: "un cliente puede cancelar **su** pedido si todavía no se envió; un vendedor puede cancelar cualquiera". Eso no entra en un atributo, porque el atributo se evalúa antes de cargar el pedido. Para eso existe la autorización basada en recursos:

```csharp
public record CancelarPedidoRequirement : IAuthorizationRequirement;

public class CancelarPedidoHandler : AuthorizationHandler<CancelarPedidoRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, CancelarPedidoRequirement requirement, Order pedido)
    {
        var esDueno = pedido.CustomerId == context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        var esStaff = context.User.IsInRole("Vendedor") || context.User.IsInRole("Admin");

        if (esStaff || (esDueno && pedido.Estado == EstadoPedido.Pendiente))
            context.Succeed(requirement);

        return Task.CompletedTask; // si no llamó a Succeed, se deniega
    }
}

builder.Services.AddSingleton<IAuthorizationHandler, CancelarPedidoHandler>();
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("CancelarPedido", p => p.AddRequirements(new CancelarPedidoRequirement()));
```

```csharp
[HttpPost("{id:int}/cancel")]
public async Task<IActionResult> Cancelar(int id, IAuthorizationService auth, CancellationToken ct)
{
    var pedido = await db.Orders.FindAsync([id], ct);
    if (pedido is null) return NotFound();

    var resultado = await auth.AuthorizeAsync(User, pedido, "CancelarPedido");
    if (!resultado.Succeeded) return Forbid();

    pedido.Cancelar();
    await db.SaveChangesAsync(ct);
    return NoContent();
}
```

La regla queda en una clase, se prueba por separado y se reutiliza en cualquier endpoint que cancele pedidos.

### 9. RBAC, ABAC y cuándo pasar de uno a otro

| | RBAC | ABAC |
| --- | --- | --- |
| Decide según | El rol del usuario | Atributos: del usuario, del recurso, del contexto |
| Ejemplo | "Los vendedores pueden ver todos los pedidos" | "Un vendedor puede ver los pedidos **de su sucursal**, en horario laboral" |
| Ventaja | Simple de entender y de auditar | Expresa reglas finas |
| Problema | **Explosión de roles** si todo se resuelve con roles nuevos (`VendedorLimaTurnoManana`) | Más complejo de implementar y de auditar |

En la práctica, la mayoría de las aplicaciones combinan las dos: roles para las acciones generales y reglas basadas en recursos (como el `CancelarPedidoHandler`) para los casos que dependen de atributos.

-----

## Ejemplo completo

El controller del problema, corregido.

`Program.cs`:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer(/* ... */);

builder.Services.AddSingleton<IAuthorizationHandler, CancelarPedidoHandler>();
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build())
    .AddPolicy("GestionPedidos", p => p.RequireRole("Vendedor", "Admin"))
    .AddPolicy("SoloAdmin", p => p.RequireRole("Admin"))
    .AddPolicy("CancelarPedido", p => p.AddRequirements(new CancelarPedidoRequirement()));

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

`OrdersController.cs`:

```csharp
[ApiController]
[Route("api/v1/orders")]
public class OrdersController(AppDbContext db, IAuthorizationService auth) : ControllerBase
{
    private string UsuarioId => User.FindFirstValue(ClaimTypes.NameIdentifier)!;

    [HttpGet("{id:int}")]
    public async Task<ActionResult<OrderDto>> Get(int id, CancellationToken ct)
    {
        var esStaff = User.IsInRole("Vendedor") || User.IsInRole("Admin");

        var pedido = await db.Orders
            .Where(o => o.Id == id && (esStaff || o.CustomerId == UsuarioId))
            .Select(o => new OrderDto(o.Id, o.Total, o.Estado.ToString()))
            .FirstOrDefaultAsync(ct);

        return pedido is null ? NotFound() : pedido;
    }

    [HttpPut("{id:int}/direccion")]
    public async Task<IActionResult> ActualizarDireccion(int id, ActualizarDireccionRequest req, CancellationToken ct)
    {
        var pedido = await db.Orders.FirstOrDefaultAsync(o => o.Id == id && o.CustomerId == UsuarioId, ct);
        if (pedido is null) return NotFound();

        pedido.CambiarDireccion(req.Direccion);
        await db.SaveChangesAsync(ct);
        return NoContent();
    }

    [HttpPost("{id:int}/cancel")]
    public async Task<IActionResult> Cancelar(int id, CancellationToken ct)
    {
        var pedido = await db.Orders.FindAsync([id], ct);
        if (pedido is null) return NotFound();

        if (!(await auth.AuthorizeAsync(User, pedido, "CancelarPedido")).Succeeded)
            return pedido.CustomerId == UsuarioId ? Forbid() : NotFound(); // no revelar pedidos ajenos

        pedido.Cancelar();
        await db.SaveChangesAsync(ct);
        return NoContent();
    }

    [Authorize(Policy = "SoloAdmin")]
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id, CancellationToken ct) { /* ... */ }
}
```

Prueba de integración que cubre BOLA (con `WebApplicationFactory` y un helper que crea un cliente autenticado como un usuario dado):

```csharp
[Fact]
public async Task Get_PedidoDeOtroCliente_Devuelve404()
{
    var pedidoDeLuis = await CrearPedidoComo("luis");
    var clienteAna = CrearClienteAutenticadoComo("ana", rol: "Cliente");

    var respuesta = await clienteAna.GetAsync($"/api/v1/orders/{pedidoDeLuis.Id}");

    Assert.Equal(HttpStatusCode.NotFound, respuesta.StatusCode);
}
```

Qué observar:

* El dueño siempre sale de `User` (la identidad autenticada), nunca de la URL ni del body.
* El filtro de la consulta **es** la autorización: un pedido ajeno simplemente no se encuentra.
* No hay `SetValues(entidad)`: cada operación tiene su DTO con solo los campos que el usuario puede cambiar.
* Borrar exige la política `SoloAdmin` en el servidor, se muestre o no el botón en el frontend.
* La *fallback policy* protege cualquier endpoint nuevo que alguien olvide marcar.
* Hay una prueba **específica** de BOLA: es el tipo de bug que vuelve con un refactor si no está cubierto.

-----

## Errores comunes

**1. Creer que `[Authorize]` alcanza.**
Qué pasa: cualquier usuario autenticado accede a los recursos de los demás (BOLA).
Por qué: `[Authorize]` verifica que haya alguien autenticado, no qué puede hacer con cada recurso.
Arreglo: en todo endpoint con un Id, verifica que el recurso pertenece al usuario o que su rol lo permite.

**2. Tomar el Id del usuario de la petición.**
Qué pasa: `?customerId=otro` o `"userId": "otro"` en el body da acceso a datos ajenos.
Por qué: se confía en un dato que el cliente controla.
Arreglo: el usuario sale siempre de los claims de la identidad autenticada.

**3. Recibir la entidad completa en un `PUT` o `POST`.**
Qué pasa: el cliente modifica precios, estados, roles o dueños (asignación masiva).
Por qué: el binding copia todos los campos que llegan.
Arreglo: un DTO de entrada por operación, con solo los campos permitidos.

**4. Ocultar en el frontend en lugar de proteger en el servidor.**
Qué pasa: endpoints de administración que cualquiera puede llamar con `curl`.
Por qué: se confunde la experiencia de usuario con la seguridad.
Arreglo: políticas en el servidor; el frontend solo refleja los permisos.

**5. Olvidar proteger un endpoint nuevo.**
Qué pasa: un endpoint queda público sin que nadie lo note.
Por qué: la protección depende de acordarse de poner `[Authorize]`.
Arreglo: *fallback policy* que exige autenticación y `[AllowAnonymous]` explícito para lo público.

**6. No tener pruebas de autorización.**
Qué pasa: un refactor quita un filtro por dueño y nadie lo detecta.
Por qué: las pruebas cubren el camino feliz con el dueño del recurso.
Arreglo: pruebas de integración con **otro** usuario intentando acceder al recurso.

-----

## Evolución y versiones

* **ASP.NET Core 2.0:** autorización basada en políticas y en recursos (`IAuthorizationService`).
* **ASP.NET Core 3.0:** `FallbackPolicy` para denegar por defecto.
* **2021:** Broken Access Control pasa al primer lugar del OWASP Top 10.
* **.NET 7:** `AddAuthorizationBuilder()` para registrar políticas de forma más concisa.
* **2023:** BOLA encabeza el OWASP API Security Top 10 por segunda edición consecutiva.

-----

## Cuándo sí y cuándo no

**Verifica el dueño del recurso cuando:** el endpoint recibe un identificador de algo que pertenece a un usuario o a una organización. Es decir, casi siempre.

**Usa roles (RBAC) cuando:** la regla depende solo del tipo de usuario: "solo administradores".

**Usa autorización basada en recursos cuando:** la regla depende del recurso concreto: dueño, estado, sucursal.

**No uses roles nuevos para cada caso particular:** si empiezas a crear `VendedorLimaTurnoManana`, la regla depende de atributos: pásala a una política basada en recursos.

-----

## Resumen en 5 líneas

1. Autenticar es saber quién es; autorizar es saber si puede hacer **esto** con **este** recurso.
2. BOLA: el dueño sale de la identidad autenticada y se usa como filtro en la consulta; para recursos ajenos, `404`.
3. Un DTO de entrada por operación evita la asignación masiva; un GUID no reemplaza la verificación.
4. RBAC con políticas con nombre para las acciones; autorización basada en recursos cuando la regla depende del recurso.
5. Denegar por defecto con una *fallback policy*, verificar todo en el servidor y probar el acceso con otro usuario.

-----

## Para profundizar

<details>
<summary>Multi-tenancy: filtros globales de consulta en EF Core</summary>

En una aplicación con varias organizaciones (*tenants*), olvidar un `.Where(x => x.TenantId == ...)` en una sola consulta expone datos de una empresa a otra. EF Core permite un filtro global que se aplica a **todas** las consultas de una entidad:

```csharp
protected override void OnModelCreating(ModelBuilder b)
{
    b.Entity<Order>().HasQueryFilter(o => o.TenantId == _tenantActual.Id);
}
```

Es defensa en profundidad: el filtro por dueño en cada consulta sigue siendo necesario para los datos de cada usuario, pero el aislamiento entre organizaciones ya no depende de acordarse.

</details>

<details>
<summary>Permisos en lugar de roles</summary>

Una alternativa a verificar roles en el código (`RequireRole("Vendedor", "Admin")`) es verificar **permisos** (`RequireClaim("permiso", "pedidos.cancelar")`) y mantener en un solo lugar (la base de datos o el proveedor de identidad) qué permisos tiene cada rol. Así, darle a un rol nuevo la capacidad de cancelar pedidos es un cambio de configuración, no de código. Es útil cuando los roles cambian seguido o los define el cliente.

</details>

-----

## En entrevista

### Respuesta corta (junior)

La autenticación verifica quién es el usuario y la autorización verifica qué puede hacer. En ASP.NET Core uso `[Authorize]` y roles como `[Authorize(Roles = "Admin")]`. BOLA es cuando un usuario puede ver recursos de otro cambiando el Id en la URL.

### Respuesta ampliada (semi-senior)

`[Authorize]` solo verifica autenticación, así que en cada endpoint que recibe un Id verifico que el recurso pertenece al usuario, tomando su Id de los claims y usándolo como filtro en la consulta; si no es suyo, devuelvo `404` para no revelar que existe. Evito la asignación masiva con un DTO de entrada por operación. Para las acciones uso RBAC con políticas con nombre, y para reglas que dependen del recurso, autorización basada en recursos con un `AuthorizationHandler`. Configuro una *fallback policy* para que todo requiera autenticación por defecto, verifico todo en el servidor aunque el frontend oculte botones, y escribo pruebas de integración donde otro usuario intenta acceder al recurso.

### Preguntas frecuentes de seguimiento

**1. ¿Qué es BOLA y cómo lo previenes?**
Acceder a un objeto ajeno cambiando su Id. Lo prevengo verificando en el servidor que el objeto pertenece al usuario autenticado, idealmente filtrando la consulta por su Id.

**2. ¿`403` o `404` si el recurso es de otro usuario?**
Normalmente `404`, para no revelar que el recurso existe y evitar la enumeración.

**3. ¿Qué es la asignación masiva?**
Que el servidor copie a la entidad todos los campos del body, incluidos los que el usuario no debería cambiar, como el precio o el rol. Se previene con DTOs de entrada específicos.

**4. ¿Usar GUIDs evita BOLA?**
No. Dificulta adivinar los Ids, pero si alguien obtiene uno y no hay verificación de dueño, accede igual.

-----

## Práctica

**Ejercicio 1.** Encuentra las vulnerabilidades de control de acceso.

```csharp
[Authorize]
[HttpGet("api/v1/invoices")]
public Task<List<Invoice>> Listar([FromQuery] int companyId) =>
    db.Invoices.Where(i => i.CompanyId == companyId).ToListAsync();

[Authorize]
[HttpPost("api/v1/users/{id}/role")]
public async Task<IActionResult> CambiarRol(int id, [FromBody] string rol)
{
    var u = await db.Users.FindAsync(id);
    u!.Rol = rol;
    await db.SaveChangesAsync();
    return Ok();
}
```

<details>
<summary>Solución</summary>

* **`Listar`, BOLA:** el `companyId` viene del query string. Cualquier usuario ve las facturas de cualquier empresa. Corrección: tomar la empresa del claim del usuario (`User.FindFirstValue("company_id")`).
* **`Listar`, exposición:** devuelve la entidad `Invoice` completa; mejor un DTO.
* **`CambiarRol`, BFLA:** cualquier usuario autenticado puede cambiar roles, incluido el suyo a `Admin`. Corrección: `[Authorize(Policy = "SoloAdmin")]`.
* **`CambiarRol`, validación:** acepta cualquier texto como rol, y si el usuario no existe falla con `NullReferenceException` en lugar de `404`.

</details>

**Ejercicio 2.** Escribe la regla de autorización para: "un comentario lo puede editar su autor durante los primeros 15 minutos; un moderador, siempre".

<details>
<summary>Solución</summary>

Depende del recurso (autor, fecha) y del rol: autorización basada en recursos.

```csharp
public record EditarComentarioRequirement : IAuthorizationRequirement;

public class EditarComentarioHandler(TimeProvider reloj)
    : AuthorizationHandler<EditarComentarioRequirement, Comentario>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext ctx, EditarComentarioRequirement req, Comentario c)
    {
        var esAutor = c.AutorId == ctx.User.FindFirstValue(ClaimTypes.NameIdentifier);
        var dentroDelPlazo = reloj.GetUtcNow() - c.CreadoEn <= TimeSpan.FromMinutes(15);

        if (ctx.User.IsInRole("Moderador") || (esAutor && dentroDelPlazo))
            ctx.Succeed(req);

        return Task.CompletedTask;
    }
}
```

Usar `TimeProvider` en lugar de `DateTime.UtcNow` permite probar la regla de los 15 minutos sin esperar.

</details>

-----

## Siguiente lección

[Inyección, XSS y sanitización](04-inyeccion-xss-y-sanitizacion.md)
