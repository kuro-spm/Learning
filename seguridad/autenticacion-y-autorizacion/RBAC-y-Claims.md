# RBAC y Claims

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

**RBAC** (*Role-Based Access Control*) es un modelo de autorización que agrupa permisos en **roles** y asigna esos roles a las usuarias, en vez de gestionar permisos uno a uno por persona. Los **claims** son la pieza más básica sobre la que se construye: pares clave-valor que describen atributos de una identidad ("rol: administradora", "plan: premium", "departamento: ventas"), y de los que un rol no es más que un caso particular.

## ¿Por qué existe?

Gestionar permisos individualmente por usuaria no escala: si tienes cien personas y necesitas cambiar quién puede aprobar facturas, no quieres tocar cien registros. Agrupar permisos en roles ("Aprobadora", "Contable", "Solo lectura") permite gestionar el acceso por grupo y reasignar personas de rol sin tocar la definición de permisos.

Pero un rol por sí solo a veces es demasiado rígido: ¿qué rol le pones a alguien que puede aprobar facturas solo hasta 1000€? Ahí es donde los claims dan un paso más allá del rol simple: en vez de (o además de) un rol, la identidad lleva atributos arbitrarios que las políticas de autorización pueden combinar con lógica propia.

> Piensa en RBAC como los carnés de un gimnasio: el carné "Premium" (el rol) te da acceso a todas las salas. Un claim sería más parecido a una pulsera con datos concretos —"clases incluidas: 10", "válida hasta: 31/12"— que permite decisiones más finas que el simple "tiene carné Premium o no".

## ¿Cuándo y para qué se usa?

- **Una app de tareas de equipo**: el rol "Gestora del proyecto" puede eliminar cualquier tarea; el rol "Miembro" solo las suyas propias.
- **Una tienda online**: un claim `PlanPremium: true` desbloquea funciones adicionales del catálogo, sin necesidad de crear un rol nuevo solo para eso.
- **Un backoffice de facturación**: un claim `LimiteAprobacion: 1000` permite que la política de autorización decida caso a caso si una persona puede aprobar una factura concreta, algo que un rol fijo no podría expresar sin explotar en variantes.

## Lo mínimo que necesitas saber

**1. `ClaimsPrincipal` y `ClaimsIdentity` en .NET**

Tras autenticar a una usuaria, ASP.NET Core representa su identidad como una colección de claims, no como un simple nombre de usuario:

```csharp
var claims = new List<Claim>
{
    new Claim(ClaimTypes.NameIdentifier, "42"),
    new Claim(ClaimTypes.Role, "Administradora"),
    new Claim("PlanPremium", "true")
};
var identity = new ClaimsIdentity(claims, "cookie");
HttpContext.User = new ClaimsPrincipal(identity);
```

**2. Restringir un endpoint por rol**

```csharp
[Authorize(Roles = "Administradora")]
[HttpDelete("productos/{id}")]
public IActionResult BorrarProducto(int id) => ...;
```

**3. Los roles son, por debajo, un claim más**

En .NET, `[Authorize(Roles = "...")]` en realidad comprueba si existe un claim de tipo `ClaimTypes.Role` con ese valor. No hay una entidad especial "rol" distinta de un claim: es la convención de nombrar así al claim que agrupa permisos.

**4. Políticas basadas en claims, para reglas que un rol no expresa**

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequierePlanPremium", policy =>
        policy.RequireClaim("PlanPremium", "true"));
});

[Authorize(Policy = "RequierePlanPremium")]
[HttpGet("informes-avanzados")]
public IActionResult InformesAvanzados() => ...;
```

Para lógica más compleja que un simple "tiene este claim", se implementa un `IAuthorizationHandler` que recibe el `ClaimsPrincipal` y decide con la lógica que haga falta (por ejemplo, comparar un `LimiteAprobacion` numérico contra el importe de la factura).

**5. RBAC vs ABAC, en una frase**

RBAC decide por el rol asignado ("es Administradora"); ABAC (*Attribute-Based Access Control*) decide combinando atributos ("es del departamento Ventas, y son las 9h-18h, y el importe es menor de 1000€"). En la práctica la mayoría de aplicaciones mezclan ambos: roles para lo grueso, claims y políticas para los matices.

## Lo que NO hace

- **No gestiona identidad** — RBAC asume que ya sabes quién es la usuaria (eso lo resuelve la autenticación); solo decide qué puede hacer una vez identificada.
- **No sustituye la comprobación de propiedad de un recurso concreto** — que un rol "Editor" permita editar documentos no dice si puede editar *este* documento en particular; eso requiere una comprobación adicional a nivel de instancia (*resource-based authorization*), no solo de rol.
- **No es gratis mantenerlo si se usa mal** — un sistema con demasiados roles hiperespecíficos ("EditoraSoloTitulo", "EditoraSoloPrecio"...) es tan difícil de gestionar como no tener roles en absoluto.

## Buenas prácticas avanzadas

- **No metas lógica de negocio en el nombre del rol** — un rol debería agrupar permisos afines, no codificar reglas ("AprobadoraHasta1000"). Cuando la regla depende de un valor variable, usa un claim con ese valor y un `IAuthorizationHandler` que lo evalúe; así la regla cambia sin tener que inventar un rol nuevo por cada umbral.
- **Evita la explosión combinatoria de roles** — si te encuentras creando un rol por cada combinación de permisos que necesitas, es la señal de que hacen falta permisos independientes y combinables (claims) en vez de más roles.
- **Combina RBAC con autorización basada en recurso para el caso "solo el dueño"** — un `IAuthorizationHandler` que reciba tanto el `ClaimsPrincipal` como el recurso concreto (`AuthorizationHandler<Requisito, Pedido>`) es la forma correcta de expresar "Editora, pero solo de sus propios pedidos", algo que ningún rol estático puede capturar por sí solo.
- **No confíes en claims que pueda inyectar el cliente** — los claims válidos son los que emitió tu servidor de identidad tras autenticar, dentro de un token firmado. Si en algún punto aceptas un rol o permiso que viene en un header o payload sin verificar su origen, cualquiera puede autoconcederse "Administradora".
- **Revisa los claims en cada request, no confíes en que el token siempre esté al día** — si el rol de una usuaria cambia (se le revoca acceso), un token de larga duración emitido antes seguirá llevando el claim antiguo; para cambios sensibles, valida contra el estado actual en servidor además de contra el token.

---

*En resumen: RBAC agrupa permisos en roles para que la gestión escale, y los claims son el bloque de construcción que permite ir más allá del rol simple cuando la regla de negocio necesita matices.*
