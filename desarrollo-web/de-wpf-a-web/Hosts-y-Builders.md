# Hosts y builders en .NET

**¿Qué es?** El patrón de arranque común a todas las aplicaciones .NET modernas: un *builder* en el que registras servicios y configuración, y un objeto final que se construye a partir de él y ejecuta la aplicación. `WebApplication.CreateBuilder(...)` es solo la variante **web** de ese patrón; existen otras para apps de consola, servicios en segundo plano e incluso librerías concretas. Esta ficha explica cómo distinguirlas cuando te las cruces.

---

## ¿Por qué existe?

Durante años, cada tipo de aplicación .NET arrancaba a su manera: las webs con un `Global.asax`, las de escritorio con `App.xaml`, las de consola con un `Main` pelado. Microsoft unificó el arranque alrededor de dos piezas que hoy están en todas partes:

- **`IServiceCollection`** — la "lista de la compra" donde registras servicios (es lo que hay detrás de `builder.Services`, sea cual sea el builder).
- **`IServiceProvider`** — el contenedor ya construido que fabrica y entrega esos servicios (ver [Inyección de dependencias](Inyeccion-de-Dependencias.md)).

Cualquier builder moderno —el de web, el de consola, el de una librería— es un envoltorio alrededor de estas dos piezas. Por eso códigos que parecen de mundos distintos comparten las mismas líneas `builder.Services.AddScoped<...>()`.

> Si vienes de WPF: es como si el arranque de `App.xaml.cs` (donde montabas tu contenedor de IoC con Prism o el toolkit de MVVM) se hubiera estandarizado para que todas las apps .NET lo hagan igual.

---

## ¿Cuándo y para qué se usa?

Te cruzarás con (al menos) tres sabores del mismo patrón:

1. **`WebApplication.CreateBuilder(args)`** — para aplicaciones web. Además del contenedor de DI, monta el servidor web (Kestrel), la configuración (`appsettings.json`), el logging y el ciclo de vida. `app.Run()` se queda escuchando peticiones hasta que pares el servidor.
2. **`Host.CreateApplicationBuilder(args)`** — el *host genérico*: lo mismo (configuración, logging, DI, ciclo de vida) pero **sin servidor web**. Es la opción idiomática para apps de consola serias y servicios en segundo plano (*workers*): por ejemplo, un proceso que cada noche genera informes o procesa una cola de correos.
3. **`new ServiceCollection()` + `BuildServiceProvider()`** — el modo manual: montas el contenedor a pelo, resuelves un servicio, lo ejecutas y el programa termina. No hay host: nadie escucha nada ni gestiona el ciclo de vida por ti.

Además, muchas librerías exponen **su propio builder** que sigue el mismo patrón: `Kernel.CreateBuilder()` en Semantic Kernel, `MauiApp.CreateBuilder()` en .NET MAUI... Por dentro envuelven un `IServiceCollection`, y por eso puedes registrar en ellos tus propios servicios junto a los de la librería.

---

## Lo mínimo que necesitas saber

**1. Los dos códigos que comparas usan el mismo contenedor**

Una app de consola que monta el contenedor a mano y una web ASP.NET Core comparten el corazón: `builder.Services` es un `IServiceCollection` en ambas. Lo que cambia es quién construye el contenedor y qué pasa después.

```csharp
// App de consola: contenedor manual, "ejecuta y termina"
var services = new ServiceCollection();
services.AddScoped<GeneradorDeInformes>();
var provider = services.BuildServiceProvider();

var tarea = provider.GetRequiredService<GeneradorDeInformes>();
await tarea.RunAsync(); // hace su trabajo y el programa acaba
```

```csharp
// App web: el host construye el contenedor y se queda escuchando
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IProductoRepository, ProductoRepository>();

var app = builder.Build();
app.MapControllers();
app.Run(); // atiende peticiones hasta que pares el servidor
```

**2. La diferencia clave es el ciclo de vida, no la "sofisticación"**

El código manual puede *parecer* más avanzado porque se ve toda la fontanería (configurar el logging a mano, llamar a `BuildServiceProvider()`, resolver el servicio con `GetRequiredService`...). En realidad es lo contrario: está haciendo **a mano** lo que `WebApplication.CreateBuilder` te da hecho. La web no es menos avanzada; es que su host asume ese trabajo por ti.

**3. El servicio que resuelves al arrancar es tu "punto de entrada"**

En una app de consola, tras construir el contenedor, pides un servicio y lo ejecutas: ese `GetRequiredService<X>()` + `RunAsync()` es el equivalente al `Main` clásico, pero con todas las dependencias ya inyectadas. En una web no haces esto: el framework resuelve los controladores por ti en cada petición.

**4. Los builders de librerías son el mismo patrón con extras**

Cuando veas `Kernel.CreateBuilder()` (Semantic Kernel) u otro builder de una librería, léelo como "un `ServiceCollection` con servicios de la librería preconfigurados". Por eso puedes mezclar en él registros propios (`AddSingleton`, `AddScoped`, métodos de extensión como `AddLogging`) exactamente igual que en `WebApplication.CreateBuilder`.

---

## Lo que NO hace

- **El contenedor manual no gestiona el ciclo de vida** — no hay `app.Run()`: si tu proceso debe seguir vivo (escuchar una cola, un temporizador), eso lo montas tú o usas el host genérico.
- **`BuildServiceProvider()` no carga configuración ni logging solos** — a diferencia de los hosts, no lee `appsettings.json` ni configura proveedores de log; por eso el código manual tiene que hacerlo explícitamente.
- **Elegir builder no cambia cómo escribes tus servicios** — una clase con sus dependencias por constructor funciona igual en consola, worker o web; solo cambia quién la registra y la resuelve.

---

## Buenas prácticas avanzadas

- **Para una consola "de verdad", prefiere el host genérico al contenedor manual** — `Host.CreateApplicationBuilder()` te da gratis `appsettings.json`, logging, variables de entorno y un apagado ordenado (espera a que las tareas terminen al recibir Ctrl+C). El `BuildServiceProvider()` manual está bien para un script pequeño; en cuanto la app tiene configuración o vida larga, estás reescribiendo el host a trozos.
- **Si construyes el contenedor a mano, libéralo** — `BuildServiceProvider()` devuelve un objeto que implementa `IDisposable`; si no lo envuelves en `await using var provider = ...`, los servicios que abren conexiones no se cierran nunca. Los hosts hacen esto por ti; el modo manual, no.
- **Activa la validación que los hosts traen de serie** — `WebApplication.CreateBuilder` valida en desarrollo que ningún singleton capture un scoped (la "dependencia cautiva" de la ficha de [Inyección de dependencias](Inyeccion-de-Dependencias.md)); un `BuildServiceProvider()` pelado **no**. Pásale las opciones para no perder esa red: `BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true, ValidateOnBuild = true })`.
- **En consola no existe la petición HTTP que delimita los `Scoped`** — si registras algo como `AddScoped` en una app de consola, se comporta casi como un singleton salvo que crees ámbitos a mano con `IServiceScopeFactory`. Antes de copiar tiempos de vida de un ejemplo web, pregúntate qué delimita "una unidad de trabajo" en tu proceso (¿una ejecución?, ¿un mensaje de la cola?).

---

*En resumen: en .NET moderno todo arranca igual —un builder donde registras servicios y algo que se construye y ejecuta—; lo que distingue una app de consola de una web no es el contenedor (es el mismo), sino quién lo construye y si el proceso termina tras su tarea o se queda escuchando.*
