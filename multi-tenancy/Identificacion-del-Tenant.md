# Identificación del Tenant

## ¿Qué es?

Es el proceso por el que la aplicación averigua, **al principio de cada petición**, a qué tenant pertenece. En inglés se le llama *tenant resolution*. Sin este paso, la aplicación no sabe de quién son los datos que debe leer o escribir.

## ¿Por qué existe?

En una app multi-tenant, la *misma* URL y el *mismo* código atienden a muchos clientes. Cuando llega una request, alguien tiene que decidir: "esta petición es de Acme, no de Globex". Ese dato condiciona todo lo que viene después (qué base de datos abrir, qué filas devolver).

> Es como la recepción de un edificio de oficinas compartido: antes de dejarte subir, recepción mira tu acreditación y decide a qué planta perteneces. Si se equivoca, acabas en la oficina de otra empresa.

Identificar mal el tenant es uno de los fallos más graves posibles: significa servir los datos de un cliente a otro. Por eso el "de quién es esta petición" se resuelve una sola vez, pronto y de forma fiable.

## ¿Cuándo y para qué se usa?

Ocurre en cada petición HTTP, normalmente en un *middleware* (una pieza que se ejecuta antes que la lógica de negocio). El resultado se guarda en el contexto de la petición (ver [Tenant Context](Tenant-Context.md)) para que el resto del código lo use sin volver a calcularlo.

## Lo mínimo que necesitas saber

**1. Por subdominio**

Cada tenant accede por su propio subdominio (`acme.miapp.com`, `globex.miapp.com`). Es la forma más visible y común en SaaS.

```csharp
// host = "acme.miapp.com"
var subdomain = host.Split('.')[0];        // "acme"
var tenant = await tenantStore.FindBySubdomainAsync(subdomain);
```

**2. Por cabecera HTTP**

El cliente envía una cabecera explícita. Muy usado en APIs consumidas por otros sistemas.

```csharp
if (request.Headers.TryGetValue("X-Tenant-Id", out var headerValue))
{
    var tenant = await tenantStore.FindByIdAsync(Guid.Parse(headerValue));
}
```

**3. Por *claim* del token (la más segura)**

Cuando el usuario inicia sesión, el `TenantId` se incrusta en su token JWT. En cada petición se lee del token ya validado, así que el usuario no puede falsearlo.

```csharp
// El claim viaja firmado dentro del token: no se puede manipular
var tenantId = user.FindFirst("tenant_id")?.Value;
```

**4. Por segmento de la ruta**

El tenant va en la propia URL: `miapp.com/acme/orders`. Sencillo, pero mezcla el tenant con el enrutado.

```csharp
// Ruta: /{tenant}/orders  → se extrae el primer segmento
var tenantSlug = routeValues["tenant"]?.ToString();   // "acme"
```

**5. Resolver pronto y fallar claro**

La resolución se hace en un middleware al inicio del *pipeline*. Si el tenant no existe o está inactivo, se corta la petición ahí mismo con un error claro, antes de tocar ningún dato.

```csharp
app.Use(async (context, next) =>
{
    var tenant = await resolver.ResolveAsync(context);
    if (tenant is null || !tenant.IsActive)
    {
        context.Response.StatusCode = 404;   // tenant desconocido → no seguir
        return;
    }
    context.Items["Tenant"] = tenant;        // disponible para el resto
    await next();
});
```

## Lo que NO hace

- **No autentica al usuario** — saber *qué* tenant es no es lo mismo que saber *quién* eres; son pasos distintos (aunque el JWT los junte).
- **No confíes en datos que el cliente controla** — un subdominio o una cabecera son manipulables; cruza siempre la identidad del tenant con lo que diga el token autenticado.
- **No debe hacerse a mitad del flujo** — si lo resuelves tarde, parte del código ya habrá corrido "sin tenant"; resuélvelo al principio.

## Buenas prácticas avanzadas

- **Reserva los subdominios peligrosos antes de abrir el registro** — si cualquier cliente puede elegir su subdominio, alguien acabará registrando `www`, `api`, `admin`, `mail` o `staging`, chocando con tu infraestructura o habilitando *phishing* ("entra en `admin.miapp.com`..."). Mantén una lista de nombres reservados y valídala en el alta; añadirla después, con tenants ya registrados, es mucho más incómodo.
- **Cachea la resolución, pero con caducidad corta e invalidación al desactivar** — resolver subdominio → tenant contra la base de datos en cada petición no escala, así que la caché es obligatoria; el peligro es el otro lado: un tenant suspendido por impago (o comprometido) debe dejar de resolver *ya*, no cuando expire la caché. Usa un TTL de segundos o invalida la entrada en el momento de desactivarlo.
- **Si un usuario puede pertenecer a varios tenants, el cambio de organización pasa por reemitir el token** — el `tenant_id` viaja firmado en el JWT, así que "cambiar de organización" no puede ser un parámetro extra que se manda junto al token viejo (sería falsificable). Diseña el selector de organización como una reemisión: el usuario elige tenant y el servidor emite un token nuevo con el claim correcto.
- **Detrás de un proxy, no te fíes del `Host` a ciegas** — la resolución por subdominio depende de la cabecera `Host` (o `X-Forwarded-Host`), y un proxy inverso mal configurado permite al cliente inyectarlas. Configura el proxy para que las fije él y la aplicación para aceptar solo los dominios esperados; si no, la "identidad" del tenant descansa en una cabecera manipulable.

---

*En resumen: identificar el tenant es la recepción de la aplicación —decide de quién es cada petición antes de dejarla pasar—, y lo más fiable es leerlo del token firmado, no de lo que el cliente diga.*
