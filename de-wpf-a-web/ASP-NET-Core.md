# ASP.NET Core

## ¿Qué es?

ASP.NET Core es el framework de Microsoft para construir aplicaciones y servicios web
con C# y .NET. Es el "motor" que recibe las peticiones HTTP que llegan por la red, las
pasa por tu código y devuelve una respuesta.

## ¿Por qué existe?

Una app de WPF tenía su propio punto de arranque (`App.xaml` y la ventana principal) y
el sistema operativo se encargaba de mostrarla. Una app web necesita algo distinto: un
programa que esté siempre escuchando en la red, que reciba peticiones de muchos clientes
y sepa repartir cada una al trozo de código correcto. ASP.NET Core es ese programa. Te
da el servidor, el enrutado, la inyección de dependencias, la configuración y la
seguridad, ya resueltos, para que tú solo escribas tu lógica.

> Si vienes de WPF: ASP.NET Core es el equivalente al `App` + el bucle de mensajes de
> Windows, pero para la web. En lugar de esperar clics de ratón, espera peticiones HTTP,
> y en lugar de abrir ventanas, devuelve páginas o datos.

## ¿Cuándo y para qué se usa?

Es la base de prácticamente cualquier cosa que hagas en web con C#: una tienda online,
el backend de una app de tareas, una API para una aplicación móvil, un panel de
administración... Todas las formas de hacer web con C# que verás más adelante
([Razor Pages y MVC](Razor-Pages-y-MVC.md), [Web API](Web-API-y-REST.md),
[Blazor](Blazor.md)) se construyen **encima** de ASP.NET Core.

## Lo mínimo que necesitas saber

**1. Todo arranca en `Program.cs`**

Aquí se configura la aplicación y se pone en marcha. Es el equivalente al arranque de tu
app de WPF, pero en lugar de mostrar una ventana, empieza a escuchar peticiones:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Registras tus servicios (ver Inyección de dependencias)
builder.Services.AddControllers();
builder.Services.AddScoped<IProductoRepository, ProductoRepository>();

var app = builder.Build();

// Configuras cómo se procesan las peticiones (ver Routing y Middleware)
app.MapControllers();

app.Run(); // se queda escuchando hasta que pares el servidor
```

**2. Se ejecuta como un servicio de larga duración, no como una ventana**

Tu app de WPF vivía mientras la ventana estuviera abierta. ASP.NET Core arranca una vez
y se queda corriendo en un servidor, atendiendo peticiones día y noche hasta que se
detiene o se despliega una versión nueva.

**3. Atiende a muchos clientes a la vez**

El framework gestiona la concurrencia por ti: procesa muchas peticiones en paralelo. Por
eso no debes guardar datos de una usuaria concreta en variables compartidas de la
aplicación —se mezclarían entre clientes—. Cada petición debe ser autosuficiente.

**4. La configuración no está en `App.config`, sino en `appsettings.json`**

Las cadenas de conexión, claves y ajustes van en `appsettings.json` (con variantes por
entorno, como `appsettings.Development.json`) y se leen mediante el sistema de
configuración integrado:

```json
{
  "ConnectionStrings": {
    "BaseDatos": "Server=...;Database=Tienda;..."
  }
}
```

**5. Trae piezas clave ya integradas**

No tienes que montar a mano la inyección de dependencias, el logging ni el enrutado:
vienen incluidos. Aprenderás a usar la [inyección de dependencias](Inyeccion-de-Dependencias.md)
y el [routing y middleware](Routing-y-Middleware.md) como parte natural del framework.

## Lo que NO hace

- **No dibuja la interfaz por sí mismo** — produce HTML o datos; quien los muestra es el navegador.
- **No elige por ti el estilo de web** — tú decides entre Razor/MVC, Web API o Blazor según el caso.
- **No mantiene estado entre peticiones automáticamente** — eso lo gestionas tú (sesión, cookies, base de datos).

## Buenas prácticas avanzadas

- **Async de arriba abajo, y jamás `.Result` o `.Wait()`** — en WPF, bloquear una `Task` congelaba tu ventana; en ASP.NET Core bloquea un hilo del *thread pool* que debería estar atendiendo a otros clientes. Bajo carga, unos cuantos `.Result` bastan para que el servidor se quede sin hilos y deje de responder (*thread pool starvation*), un problema que no aparece en desarrollo con un solo usuario. Si un método toca red o disco, es `async` y se espera con `await` hasta arriba del todo.
- **Los secretos no van en `appsettings.json`** — ese archivo acaba en el repositorio. Las cadenas de conexión y claves reales van fuera: `dotnet user-secrets` en desarrollo y variables de entorno (o un almacén de secretos) en producción. El sistema de configuración los fusiona solo, así que tu código no cambia: sigue leyendo `Configuration.GetConnectionString(...)` sin saber de dónde salió el valor.
- **Registra logs con plantillas, no con interpolación** — `_logger.LogInformation("Pedido {PedidoId} creado", id)` en vez de `$"Pedido {id} creado"`. Parece un capricho, pero la plantilla convierte `PedidoId` en un campo estructurado por el que luego puedes filtrar y buscar en las herramientas de logs; la cadena interpolada es solo texto plano. Es la diferencia entre "buscar a mano en un fichero" y "consultar tus logs como una base de datos".
- **El comportamiento debe depender del entorno** — ASP.NET Core sabe si corre en `Development` o `Production` (variable `ASPNETCORE_ENVIRONMENT`). Úsalo: la página de excepciones con detalles (`UseDeveloperExceptionPage`) solo en desarrollo —en producción filtra información interna a cualquiera— y datos de prueba o configuraciones relajadas, igual. Ese `if (app.Environment.IsDevelopment())` del arranque no es decorativo.
- **Config tipada con el patrón *Options*** — en vez de leer `Configuration["Pagos:ApiKey"]` con cadenas mágicas repartidas por el código, define una clase `PagosOptions`, vincúlala una vez (`builder.Services.Configure<PagosOptions>(...)`) e inyéctala como `IOptions<PagosOptions>`. Ganas tipado, autocompletado y un único lugar donde ver qué configuración necesita cada pieza.

---

*En resumen: ASP.NET Core es el motor web de C# —el "App" de tu aplicación web— que escucha peticiones HTTP, las reparte a tu código y devuelve respuestas, con todas las piezas básicas ya integradas.*
