# [Authorize] y [AllowAnonymous]

## ¿Qué es?

`[Authorize]` es el atributo que protege un endpoint: al ponerlo, solo se puede acceder a ese método (o a todo el controlador) si la petición está autenticada, y opcionalmente si cumple ciertos roles o políticas. `[AllowAnonymous]` es su contrario: abre una excepción para dejar pasar a cualquiera. Ambos viven en `using Microsoft.AspNetCore.Authorization;`.

## ¿Por qué existe?

Casi ninguna aplicación es completamente pública: hay zonas que solo puede ver quien ha iniciado sesión, y acciones que solo puede hacer cierto perfil. Sin `[Authorize]`, tendrías que comprobar la identidad y los permisos a mano al principio de cada método. El atributo declara la regla de acceso **encima** del endpoint y deja que el framework la haga cumplir antes de que tu código se ejecute.

> Es la distinción entre autenticación ("¿quién eres?") y autorización ("¿qué puedes hacer?"). Si quieres el fondo del asunto —roles, claims, políticas—, la guía de [RBAC y Claims](../../seguridad/autenticacion-y-autorizacion/RBAC-y-Claims.md) lo desarrolla.

## ¿Cuándo y para qué se usa?

En cualquier parte protegida: el endpoint que devuelve "mis pedidos" (necesita saber quién eres), el panel de administración (solo rol `Admin`), la acción de borrar un recurso (solo su dueño o un moderador). Y `[AllowAnonymous]` para los pocos huecos públicos dentro de una zona por lo demás cerrada: el login, el registro, una página de estado.

## Lo mínimo que necesitas saber

**1. `[Authorize]` exige estar autenticado**

```csharp
[Authorize]
[HttpGet("orders")]
public IActionResult MyOrders() { /* solo si hay sesión válida */ }
```

**2. Puesto en la clase, protege todas sus acciones**

```csharp
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase { /* todo protegido */ }
```

**3. Restringe por rol o por política**

```csharp
[Authorize(Roles = "Admin")]                 // solo el rol Admin
public IActionResult DeleteProduct(int id) { }

[Authorize(Policy = "MayoresDeEdad")]        // una política definida en Program.cs
public IActionResult BuyAlcohol() { }
```

**4. `[AllowAnonymous]` abre una excepción, incluso dentro de una clase protegida**

```csharp
[Authorize]
public class AccountController : ControllerBase
{
    [AllowAnonymous]                         // este sí es público
    [HttpPost("login")]
    public IActionResult Login(LoginRequest request) { }
}
```

**5. Necesita el middleware de autenticación y autorización activo**

El atributo no funciona solo: en `Program.cs` deben estar `app.UseAuthentication()` y `app.UseAuthorization()`, y en ese orden (primero saber quién eres, luego si puedes). Si faltan, `[Authorize]` no protege nada o falla.

## Lo que NO hace

- **No autentica** — no hace login ni valida contraseñas; asume que otro mecanismo (cookies, JWT...) ya estableció la identidad. Solo comprueba el resultado.
- **No define los roles ni las políticas** — solo los referencia; se configuran en el arranque de la app.
- **No distingue al dueño del recurso** — `[Authorize]` sabe "es un usuario válido con rol X", pero "¿es *su* pedido?" es autorización basada en recurso, que va en código.

## Buenas prácticas avanzadas

- **`401` y `403` no son lo mismo** — `[Authorize]` devuelve `401 Unauthorized` cuando *no sabe quién eres* (falta o caducó la credencial) y `403 Forbidden` cuando *sabe quién eres pero no tienes permiso*. Confundirlos lleva a mensajes engañosos: un `403` no se arregla volviendo a iniciar sesión. Devolver el correcto ayuda a los clientes a reaccionar bien.
- **`[AllowAnonymous]` siempre gana: revísalo con lupa** — si una acción tiene a la vez `[Authorize]` y `[AllowAnonymous]` (heredado de la clase, por ejemplo), *pasa* como pública. Es la causa número uno de endpoints "protegidos" que en realidad están abiertos. Audita dónde pones `[AllowAnonymous]`.
- **Cierra por defecto, abre por excepción** — en vez de acordarte de poner `[Authorize]` en cada controlador (y olvidarlo en alguno), configura una *fallback policy* que exija autenticación en toda la app y usa `[AllowAnonymous]` solo donde de verdad quieras abrir. Es la diferencia entre "seguro salvo que lo olvide" y "seguro salvo que lo permita explícitamente".
- **Prefiere políticas a roles para reglas finas** — `Roles = "Admin"` se queda corto en cuanto la regla es "mayor de edad", "del mismo departamento" o "con permiso de facturación". Las *policies* basadas en claims expresan eso sin ensuciar los controladores con lógica, y se cambian en un único sitio. Ver [RBAC y Claims](../../seguridad/autenticacion-y-autorizacion/RBAC-y-Claims.md).

---

*En resumen: `[Authorize]` declara "para entrar aquí hay que estar autenticado (y cumplir estos roles o políticas)" y `[AllowAnonymous]` abre huecos públicos — la regla de acceso vive encima del endpoint y el framework la aplica antes de tu código.*
