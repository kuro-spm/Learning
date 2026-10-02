# IP real del cliente detrás de un reverse proxy

## ¿Qué es?

Cuando una API ASP.NET Core vive detrás de un reverse proxy, la dirección que ve en la conexión TCP es la del proxy, no la de la persona que hace la petición. «Recuperar la IP real» consiste en leerla de las cabeceras `X-Forwarded-*` que añade el proxy, **confiando solo en las que vienen de un proxy que tú has declarado**.

## ¿Por qué existe?

Con el despliegue habitual (nginx delante, la API en un contenedor detrás), toda petición llega a la API desde la misma IP: la del proxy. Mira qué ve el código:

```csharp
var ip = context.Connection.RemoteIpAddress;   // 172.18.0.2, la del proxy, para TODOS los clientes
```

Para una tienda online esto tiene dos consecuencias graves en el endpoint `/api/auth/login`, si limitas los intentos por IP (ver [Limitar intentos de login](Limitar-intentos-de-login.md)):

- **Todos los clientes comparten un único cubo.** Con un límite de 10 intentos por minuto, basta que 10 intentos fallidos ocurran entre todos los usuarios para que el login devuelva `429` a todo el mundo. Es un DoS del login causado por el propio límite.
- **La fuerza bruta no tiene freno por IP.** Si para evitar lo anterior subes el límite hasta que nadie lo alcance, quien ataca desde una sola IP prueba contraseñas sin que nada lo distinga de los demás.

Lo mismo pasa con los logs de auditoría (todas las líneas dicen «IP del proxy») y con herramientas como [Fail2ban](../../devops/despliegue-en-vps/Fail2ban.md), que acabaría bloqueando al proxy.

La solución es que el proxy cuente a la API quién es el cliente, y que la API lo crea **solo cuando quien se lo cuenta es el proxy**. Esa segunda mitad es la que casi todo el mundo se salta, y es la que convierte la solución en un agujero.

> Si has usado `request.getRemoteAddr()` en Java detrás de un balanceador, es el mismo problema: la solución allí también son las cabeceras `X-Forwarded-*` y una lista de proxies de confianza.

## ¿Cuándo y para qué se usa?

Siempre que la API no reciba la conexión del navegador directamente: nginx o [nginx-proxy](../../devops/despliegue-en-vps/Reverse-Proxy-con-nginx-proxy.md) en un VPS, un balanceador de la nube, un ingress de Kubernetes, un CDN. Se nota en cuatro sitios: el límite de peticiones por IP, los logs, la geolocalización o reglas por país, y la redirección a HTTPS (el esquema real también viaja en una cabecera).

La ficha [Despliegue de una app ASP.NET Core](../../desarrollo-web/asp-net-core/Despliegue.md) lo menciona como buena práctica; aquí está el detalle de cómo configurarlo sin abrir un bypass.

---

## Las cabeceras que añade el proxy

El proxy reenvía la petición en una conexión nueva y deja constancia del origen en cabeceras HTTP:

| Cabecera | Contenido | Ejemplo |
|---|---|---|
| `X-Forwarded-For` | Lista de IPs por las que ha pasado la petición, separadas por coma | `203.0.113.7` |
| `X-Forwarded-Proto` | Esquema con el que llegó al proxy | `https` |

Aunque no es un estándar oficial, es el uso universal. El estándar formal es la cabecera `Forwarded` ([RFC 7239](https://www.rfc-editor.org/rfc/rfc7239)), pero en la práctica casi todo el mundo habla `X-Forwarded-*`, y es lo que ASP.NET Core procesa por defecto cuando se lo pides.

En nginx se configura dentro del bloque `location` que hace el `proxy_pass`:

```nginx
location /api/ {
    proxy_pass http://api:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

La variable `$proxy_add_x_forwarded_for` hace algo concreto: toma el `X-Forwarded-For` que ya traiga la petición del cliente y **le añade al final la IP que nginx ve en la conexión** (`$remote_addr`). Si no traía la cabecera, queda solo esa IP. Es decir, **conserva lo que mandó el cliente, y eso incluye lo falsificado**:

| El cliente envía | La API recibe |
|---|---|
| (nada) | `X-Forwarded-For: 203.0.113.7` |
| `X-Forwarded-For: 1.2.3.4` | `X-Forwarded-For: 1.2.3.4, 203.0.113.7` |

La IP verdadera es siempre la **última**: es la que añadió nuestro proxy, y el cliente no puede escribir después de ella. Todo lo anterior lo puede haber inventado cualquiera.

## Leer las cabeceras en ASP.NET Core

El middleware `UseForwardedHeaders` lee esas cabeceras y **reescribe** `Connection.RemoteIpAddress` y `Request.Scheme` con los valores reales. El resto del pipeline (limitador, logs, `Request.IsHttps`) ya ve la IP del cliente sin enterarse de que hubo un proxy.

La versión ingenua, que no deberías desplegar:

```csharp
// ❌ Confía en cualquiera que mande la cabecera
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    o.KnownProxies.Clear();
    o.KnownIPNetworks.Clear();      // .NET 10; en versiones anteriores: KnownNetworks.Clear()
});
```

Con esto se acepta la cabecera venga de donde venga. Es el bypass de manual: quien ataca llama a la API directamente (o a través del proxy, que conserva su cabecera) y rota el valor en cada intento:

```bash
curl -X POST https://tienda.ejemplo.com/api/auth/login \
  -H "X-Forwarded-For: 10.20.30.1" -d '{"email":"ana@ejemplo.com","password":"intento-1"}'
curl -X POST https://tienda.ejemplo.com/api/auth/login \
  -H "X-Forwarded-For: 10.20.30.2" -d '{"email":"ana@ejemplo.com","password":"intento-2"}'
```

Cada petición parece venir de una IP distinta, así que cada una cae en un cubo nuevo del limitador y el límite no se alcanza nunca. Un control de seguridad que se esquiva cambiando un número de una cabecera no es un control.

### La versión segura

Se declara quién es el proxy, y solo entonces se aceptan sus cabeceras:

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    o.ForwardLimit = 1;                                   // solo un salto de confianza

    o.KnownProxies.Clear();                               // descarta el loopback que viene de serie
    o.KnownProxies.Add(IPAddress.Parse("10.0.0.5"));      // IP exacta del proxy

    o.KnownIPNetworks.Clear();
    o.KnownIPNetworks.Add(System.Net.IPNetwork.Parse("172.18.0.0/16"));   // o la red completa
});

var app = builder.Build();
app.UseForwardedHeaders();      // la primera línea del pipeline
```

Las opciones, una a una:

| Opción | Qué hace |
|---|---|
| `ForwardedHeaders` | Qué cabeceras se procesan. Por defecto vale `None`: **sin ponerla, el middleware no hace nada** |
| `KnownProxies` | IPs exactas de proxies de confianza |
| `KnownIPNetworks` | Redes en notación CIDR (`172.18.0.0/16`) de proxies de confianza. Hasta .NET 9 se llama `KnownNetworks` y usa su propio tipo `IPNetwork` de `Microsoft.AspNetCore.HttpOverrides`; en .NET 10 la propiedad antigua está obsoleta |
| `ForwardLimit` | Cuántas entradas de `X-Forwarded-For` se procesan, contando desde la derecha. Por defecto 1 |

### Qué entrada se lee exactamente

El middleware recorre `X-Forwarded-For` **de derecha a izquierda**, con esta regla: «si la IP desde la que me llegó la petición es un proxy conocido, me creo la entrada que me ha puesto, y esa pasa a ser la nueva IP del cliente».

- Si la conexión **no** viene de un proxy conocido, se ignora la cabecera entera. La IP sigue siendo la de la conexión.
- Si viene de un proxy conocido, se toma la última entrada. Con `ForwardLimit = 1` el proceso se detiene ahí.

El caso de varias entradas falsificadas, con nginx como único proxy y `ForwardLimit = 1`:

```
Cliente real: 203.0.113.7  →  nginx (10.0.0.5)  →  API

El cliente manda:   X-Forwarded-For: 1.1.1.1, 2.2.2.2
nginx lo deja en:   X-Forwarded-For: 1.1.1.1, 2.2.2.2, 203.0.113.7

La API lee (1 salto):  203.0.113.7    ← la que añadió nginx
Se descartan:          1.1.1.1, 2.2.2.2  ← inventadas, ignoradas
```

Falsificar entradas **antes** de la real no sirve de nada, porque con `ForwardLimit = 1` ni se leen. Es el ajuste correcto cuando hay exactamente un proxy.

Con más proxies en cadena, hay que subir el límite al número de saltos de confianza, y declarar a **todos** como proxies conocidos. Si pones el límite más alto que ese número, empiezas a leer entradas que escribió el cliente.

## Configurarlo «seguro por defecto»

Hay tres decisiones de diseño que evitan que un descuido de configuración reabra el bypass:

1. **La lista de proxies viene de la configuración y vacía significa «no confío en nadie».** En desarrollo y en tests no hay proxy: no hace falta cambiar nada y no se acepta ninguna cabecera.
2. **El middleware solo se activa si hay configuración.** No se registra un `UseForwardedHeaders` con la lista por defecto «por si acaso».
3. **La configuración se valida al arrancar.** Una IP mal escrita no debe ignorarse en silencio: dejaría la app sin proxies de confianza (el cubo compartido de antes) sin que nadie lo note. Mejor que no arranque.

```csharp
public sealed class ForwardedHeadersSettings
{
    public const string SectionName = "ForwardedHeaders";
    public string[] KnownProxies { get; set; } = [];
    public string[] KnownNetworks { get; set; } = [];     // CIDR, p. ej. "172.18.0.0/16"
    public int ForwardLimit { get; set; } = 1;
}

public static class ForwardedHeadersSetup
{
    public static ForwardedHeadersOptions? CrearOpciones(IConfiguration configuration)
    {
        var settings = configuration.GetSection(ForwardedHeadersSettings.SectionName)
            .Get<ForwardedHeadersSettings>() ?? new();

        if (settings.KnownProxies.Length == 0 && settings.KnownNetworks.Length == 0)
            return null;                                   // sin configuración: no se confía en nada

        if (settings.ForwardLimit < 1)
            throw new InvalidOperationException("ForwardedHeaders:ForwardLimit debe ser >= 1.");

        var opciones = new ForwardedHeadersOptions
        {
            ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto,
            ForwardLimit = settings.ForwardLimit,
        };
        opciones.KnownProxies.Clear();
        opciones.KnownIPNetworks.Clear();

        foreach (var proxy in settings.KnownProxies)
        {
            if (!IPAddress.TryParse(proxy, out var ip))
                throw new InvalidOperationException($"ForwardedHeaders:KnownProxies contiene una IP no válida: '{proxy}'.");
            opciones.KnownProxies.Add(ip);
        }

        foreach (var red in settings.KnownNetworks)
        {
            if (!red.Contains('/') || !System.Net.IPNetwork.TryParse(red, out var network))
                throw new InvalidOperationException($"ForwardedHeaders:KnownNetworks contiene un CIDR no válido: '{red}' (formato: 10.0.0.0/8).");
            opciones.KnownIPNetworks.Add(network);
        }

        return opciones;
    }
}
```

Y en `Program.cs`:

```csharp
var forwardedHeaders = ForwardedHeadersSetup.CrearOpciones(builder.Configuration);
// ...
if (forwardedHeaders is not null)
    app.UseForwardedHeaders(forwardedHeaders);
```

En producción, la configuración sale de variables de entorno (los arrays se numeran con `__`):

```yaml
services:
  api:
    environment:
      - ForwardedHeaders__KnownNetworks__0=172.18.0.0/16
      - ForwardedHeaders__ForwardLimit=1
```

> Si usas versiones anteriores a .NET 10, la lógica es la misma: cambia `KnownIPNetworks` por `KnownNetworks`, y crea cada red con `new IPNetwork(IPAddress.Parse("172.18.0.0"), 16)`, ya que ese tipo no tiene `Parse` y recibe prefijo y longitud por separado.

## Orden de middlewares

`UseForwardedHeaders` reescribe la IP y el esquema para el resto, así que tiene que ir **antes de todo lo que los lea**:

```csharp
app.UseForwardedHeaders(forwardedHeaders);   // 1.º: arregla IP y esquema
app.UseHttpsRedirection();                   // necesita el esquema real
app.UseRateLimiter();                        // particiona por IP real
// logging de peticiones, autenticación...
```

Dos fallos típicos por ponerlo tarde:

- **Limitador antes que el middleware.** El limitador ya partició por la IP del proxy: el cubo compartido sigue ahí aunque hayas configurado todo bien.
- **`UseHttpsRedirection` antes que el middleware: bucle de redirecciones.** El proxy termina el TLS y habla HTTP con la API, así que sin `X-Forwarded-Proto` la API cree que la petición es HTTP, responde `307` a la versión HTTPS, el navegador vuelve a pedirla, el proxy la reenvía como HTTP... y el navegador acaba con `ERR_TOO_MANY_REDIRECTS`.

La única excepción es el manejo de errores global (`UseExceptionHandler`), que se suele poner aún antes.

## Probarlo con `WebApplicationFactory`

Con `WebApplicationFactory` no hay socket: el `TestServer` deja `RemoteIpAddress` a `null`, así que no hay «IP del proxy» que el middleware pueda comparar con la lista. Hay que fijarla a mano con un `IStartupFilter` de test, que se ejecuta antes que cualquier middleware de la app y toma la IP «de conexión» de una cabecera inventada:

```csharp
private sealed class RemoteIpDeCabeceraFilter : IStartupFilter
{
    public Action<IApplicationBuilder> Configure(Action<IApplicationBuilder> next) => app =>
    {
        app.Use((context, nextMiddleware) =>
        {
            if (context.Request.Headers.TryGetValue("X-Test-Remote-Ip", out var ip))
                context.Connection.RemoteIpAddress = IPAddress.Parse(ip.ToString());
            return nextMiddleware(context);
        });
        next(app);
    };
}

private static WebApplicationFactory<Program> CrearApp(params (string Clave, string Valor)[] config) =>
    new WebApplicationFactory<Program>().WithWebHostBuilder(b =>
    {
        foreach (var (clave, valor) in config)
            b.UseSetting(clave, valor);
        b.ConfigureTestServices(s => s.AddTransient<IStartupFilter, RemoteIpDeCabeceraFilter>());
    });
```

`UseSetting` inyecta la configuración (`ForwardedHeaders:KnownProxies:0`) sin tocar ficheros. El helper `LoginAsync(client, ipConexion, forwardedFor)` envía un `POST /api/auth/login` con `X-Test-Remote-Ip` y, opcionalmente, `X-Forwarded-For`. Para observar el cubo basta hacer 10 peticiones (el límite) y comprobar que la 11.ª recibe `429`.

Los cinco casos que merece la pena cubrir, con la configuración `KnownProxies = 10.0.0.5` y límite 10 por minuto:

```csharp
[Fact]
public async Task ProxyConfiado_DosClientesDistintos_NoComparten_Cubo()
{
    await using var app = CrearApp(("ForwardedHeaders:KnownProxies:0", "10.0.0.5"));
    var client = app.CreateClient();

    await AgotarCuboAsync(client, "10.0.0.5", "203.0.113.1");

    Assert.Equal(HttpStatusCode.TooManyRequests, await LoginAsync(client, "10.0.0.5", "203.0.113.1"));
    Assert.Equal(HttpStatusCode.BadRequest,      await LoginAsync(client, "10.0.0.5", "203.0.113.2"));
}

[Fact]
public async Task IpNoConfiada_IgnoraXForwardedFor()
{
    await using var app = CrearApp(("ForwardedHeaders:KnownProxies:0", "10.0.0.5"));
    var client = app.CreateClient();

    // Quien llama directamente rota la cabecera para esquivar el límite
    for (var i = 0; i < 10; i++)
        await LoginAsync(client, "198.51.100.7", $"203.0.113.{i + 1}");

    Assert.Equal(HttpStatusCode.TooManyRequests, await LoginAsync(client, "198.51.100.7", "203.0.113.200"));
}
```

Qué hace cada uno y qué devuelve:

| Caso | Se comprueba | Resultado esperado |
|---|---|---|
| Proxy confiado, dos clientes | La IP de cada cliente se usa como clave | Cliente 1 agotado → `429`; cliente 2 → no afectado |
| Entradas falsificadas antes de la real | Con `ForwardLimit = 1` solo cuenta la última | `1.1.1.1, 203.0.113.1` y `9.9.9.9, 203.0.113.1` caen en el mismo cubo |
| Conexión desde IP no confiada | La cabecera se ignora entera | Rotar `X-Forwarded-For` no esquiva el límite |
| Sin configuración | No se confía en nada, ni siquiera desde la IP del proxy | Mismo resultado que el caso anterior |
| IP o CIDR inválidos | La app no arranca | Excepción cuyo mensaje contiene `ForwardedHeaders` |

El último caso se prueba con un `[Theory]` que pasa valores como `no-es-una-ip`, `10.0.0.0` (sin `/`) o `10.0.0.0/99`, y verifica que `app.CreateClient()` lanza una excepción.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `RemoteIpAddress` sigue siendo la del proxy | Falta `UseForwardedHeaders`, o `ForwardedHeaders` quedó en `None`, o la IP del proxy no está en `KnownProxies`/`KnownIPNetworks` y por eso se ignora la cabecera |
| En Docker no funciona aunque pusiste la IP del host | La API ve la IP del contenedor del proxy en la red de Docker (`172.x.x.x`), no la del servidor. Declara esa red como CIDR (`docker network inspect <red>` muestra su subred) |
| Funciona en local pero no en producción (o al revés) | Los defaults de ASP.NET Core confían solo en el loopback (`127.0.0.1` y `::1`). En local con un proxy en la misma máquina funciona; en producción no, hasta que declaras al proxy real |
| `KnownIPNetworks`/`KnownProxies` incluye `0.0.0.0/0` | Confía en todo internet: es el bypass de la versión ingenua con otro nombre |
| Un solo proxy con `ForwardLimit` alto (o `null`) | Se leen entradas que escribió el cliente. Fija el límite al número exacto de proxies de confianza |
| Varios proxies encadenados y solo declaras el último | La cadena se detiene en el primero no declarado. Declara cada salto y ajusta `ForwardLimit` |
| `ERR_TOO_MANY_REDIRECTS` | Falta `X-Forwarded-Proto` en el proxy, o `UseHttpsRedirection` va antes del middleware |
| Todo bien en dev, activaste la confianza global «para que funcione» | Ver abajo: se pone en producción algo que solo hacía falta en un entorno |

Con una **cadena de proxies** (por ejemplo un CDN como Cloudflare delante de tu nginx), la IP del cliente viaja en la cabecera, pero ahora hay dos saltos de confianza: nginx ve la IP del CDN, la API ve la de nginx. Eso exige `ForwardLimit = 2` y declarar a ambos como conocidos. Los CDN además publican los rangos de IP desde los que te llaman, que cambian con el tiempo: revísalos en su documentación en lugar de copiarlos una vez.

Una nota sobre la variable de entorno `ASPNETCORE_FORWARDEDHEADERS_ENABLED=true`. Activa el middleware sin escribir código, pero para que funcione en contenedores **limpia las listas de proxies de confianza**, es decir, acepta las cabeceras de cualquiera. Es cómoda en una plataforma donde nada llega a tu contenedor sin pasar por el balanceador; si el contenedor es alcanzable por otra vía, es el bypass de antes.

## Buenas prácticas avanzadas

- **Haz que el proxy, además, sobrescriba la cabecera en vez de acumularla.** Si tu nginx está directamente de cara a internet, usar `proxy_set_header X-Forwarded-For $remote_addr;` (en lugar de `$proxy_add_x_forwarded_for`) descarta lo que mandó el cliente. Así la API recibe una sola IP y el límite no depende de `ForwardLimit`. Solo es válido si **no** hay otro proxy delante cuya cabecera necesitas conservar.
- **Prefiere la red (CIDR) a una IP suelta cuando el proxy está en un contenedor.** La IP de un contenedor en una red de Docker puede cambiar al recrearlo. Una subred dedicada a la comunicación proxy-API (`172.18.0.0/16`) es estable y no abre la puerta a nadie de fuera de ella.
- **Mantén la subred de confianza lo más pequeña posible.** Si en la misma red de Docker hay otros contenedores (un servicio de terceros, un worker), también serán «proxies de confianza» y podrán falsificar su IP. Una red solo para el proxy y la API lo evita.
- **No dejes que la API sea alcanzable sin pasar por el proxy.** Si el puerto de la API está publicado en el host (`ports: "8080:8080"`), cualquiera puede saltarse el proxy. La confianza en `X-Forwarded-For` descansa en que el único que llega a la API desde la red de confianza es el proxy; el [firewall](../../devops/despliegue-en-vps/UFW.md) y no publicar puertos son parte de la solución.
- **Cuenta cada intento por IP *y* por cuenta.** Por muy bien que recuperes la IP, quien dispone de una botnet usa muchas distintas. Un segundo límite por email (o por usuario) frena el ataque distribuido contra una sola cuenta, que el límite por IP no ve.
- **Registra la IP original además de la reescrita en los logs de seguridad.** Tras `UseForwardedHeaders`, ASP.NET Core guarda los valores originales en las cabeceras `X-Original-For` y `X-Original-Proto`; guardar ambos en un incidente permite ver qué se falsificó.

## Documentación oficial

- [Configure ASP.NET Core to work with proxy servers and load balancers](https://learn.microsoft.com/aspnet/core/host-and-deploy/proxy-load-balancer) — la sección «Forwarded Headers Middleware» y la de «Troubleshoot» son las que importan; ahí están los valores por defecto y los escenarios con varios proxies.
- [X-Forwarded-For en MDN](https://developer.mozilla.org/docs/Web/HTTP/Headers/X-Forwarded-For) — explica el formato de la cabecera y avisa de la falsificación de entradas; trae la advertencia de seguridad que justifica toda esta ficha.
- [RFC 7239: Forwarded HTTP Extension](https://www.rfc-editor.org/rfc/rfc7239) — el estándar que formaliza la cabecera `Forwarded`, para cuando la duda es qué dice la especificación y no la práctica.

## Recursos didácticos

- [WebApplicationFactory en la guía de tests de integración](https://learn.microsoft.com/aspnet/core/test/integration-tests) — para montar el `TestServer` con configuración propia, como en los casos de arriba.

---

*En resumen: detrás de un proxy la IP que ve la API es la del proxy, así que se lee de `X-Forwarded-For` — pero solo se acepta la cabecera de un proxy declarado, solo la entrada que él añadió y solo si la configuración vacía significa «no confío en nadie».*
