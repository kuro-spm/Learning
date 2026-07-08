# De .NET Core a ASP.NET Core

## ¿Qué es?

ASP.NET Core es el framework de Microsoft para construir aplicaciones y servicios web sobre .NET. Si .NET Core es el runtime y las librerías que ya usas para hacer una app de consola o una librería, **ASP.NET Core es lo que se monta encima para que ese mismo .NET escuche peticiones HTTP y responda por la red**.

## ¿Por qué existe?

Una app de consola de .NET Core arranca en `Main`, hace su trabajo y termina. Una aplicación web necesita algo distinto: un programa que **nunca termina**, que está permanentemente escuchando en un puerto, que recibe peticiones de muchos clientes a la vez y sabe repartir cada una al trozo de código correcto.

Podrías montar todo eso a mano sobre .NET Core (abrir un socket, parsear HTTP, gestionar hilos...), pero es justo lo que ASP.NET Core te da ya resuelto: el servidor, el enrutado, la inyección de dependencias, la configuración, el logging y la seguridad. Tú solo escribes tu lógica.

> Piénsalo así: ASP.NET Core es una app de consola de .NET que, en lugar de hacer una tarea y salir, se queda en un bucle infinito atendiendo peticiones HTTP. Todo lo que ya sabes de C# y .NET sigue valiendo; solo cambia el "qué dispara tu código".

## ¿Cuándo y para qué se usa?

Es la base de prácticamente cualquier cosa web que hagas con C#: el backend de una tienda online, una API para una app móvil, un panel de administración, un servicio interno que otros consumen. Cuando en un proyecto ves un `Program.cs` que llama a `WebApplication.CreateBuilder`, estás ante una app ASP.NET Core.

## Lo mínimo que necesitas saber

Estas son las diferencias clave respecto a una app de consola de .NET Core que ya dominas.

**1. Sigue arrancando en `Program.cs`, pero con un *builder* web**

El patrón *builder* es el mismo que el de los hosts genéricos de .NET; lo específico aquí es `WebApplication`:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();                              // registras servicios
builder.Services.AddScoped<IProductRepository, ProductRepository>();

var app = builder.Build();

app.MapControllers();                                          // configuras el pipeline
app.Run();                                                     // se queda escuchando; NO retorna
```

La diferencia con una consola: `app.Run()` no termina. Se queda atendiendo peticiones hasta que pares el proceso.

**2. Trae un servidor web integrado: Kestrel**

No necesitas IIS ni Apache para arrancar. ASP.NET Core incluye **Kestrel**, un servidor HTTP propio y multiplataforma que escucha en un puerto y entrega cada petición a tu pipeline. En producción suele ir detrás de un *reverse proxy* (Nginx, IIS, un balanceador), pero para desarrollar y ejecutar te basta con Kestrel.

**3. La inyección de dependencias viene de serie**

En una consola tendrías que traerte un contenedor de DI o instanciar todo a mano. Aquí el contenedor está integrado: registras servicios en `builder.Services` con un tiempo de vida (`AddSingleton`, `AddScoped`, `AddTransient`) y los recibes por constructor donde los necesites.

```csharp
public class ProductsController(IProductRepository repo) : ControllerBase
{
    // 'repo' llega inyectado; no lo instancias tú
}
```

**4. Configuración y logging, también integrados**

`appsettings.json` (con variantes por entorno como `appsettings.Development.json`), variables de entorno y secretos se fusionan solos en un `IConfiguration`. Y hay un sistema de logging (`ILogger<T>`) listo para inyectar. No montas nada: ya está cableado en el builder.

**5. El pipeline de middleware y el routing deciden qué se ejecuta**

En una consola, el flujo lo marca tu código de arriba abajo. En ASP.NET Core, cada petición atraviesa una **tubería de middleware** (autenticación, errores, logging...) y el **routing** decide qué método la atiende según su URL y su verbo. Es un cambio de mentalidad: tu código deja de "llamarse a sí mismo" y pasa a "ser llamado" por el framework cuando llega la petición que le corresponde.

**6. El estado por defecto es "por petición", no global**

Como atiende a muchos clientes en paralelo, no puedes guardar datos de un cliente en variables estáticas: se mezclarían entre peticiones. Por eso existe el tiempo de vida `Scoped`: una instancia nueva por cada petición. Cada petición debe ser autosuficiente.

## Lo que NO hace

- **No es un lenguaje ni un runtime nuevo** — sigues escribiendo C# sobre .NET; ASP.NET Core es una capa de librerías encima.
- **No dibuja la interfaz por sí mismo** — produce HTML o datos (JSON); quien lo muestra es el navegador o el cliente.
- **No mantiene estado entre peticiones automáticamente** — eso lo gestionas tú (sesión, cookies, base de datos).
- **No te obliga a un único estilo** — puedes hacer APIs con controladores, *minimal APIs*, páginas Razor/MVC o Blazor; todo sobre el mismo motor.

## Buenas prácticas avanzadas

- **Async de arriba abajo, y jamás `.Result` o `.Wait()`** — en una consola con un solo usuario, bloquear una `Task` no se nota. En ASP.NET Core bloquea un hilo del *thread pool* que debería estar atendiendo a otros clientes; bajo carga, unos cuantos `.Result` bastan para dejar el servidor sin hilos (*thread pool starvation*), un fallo que no aparece en desarrollo. Si un método toca red o disco, es `async` y se espera con `await` hasta arriba.
- **Los secretos no van en `appsettings.json`** — ese archivo acaba en el repositorio. Cadenas de conexión y claves reales van fuera: `dotnet user-secrets` en desarrollo y variables de entorno (o un almacén de secretos) en producción. El sistema de configuración los fusiona solo, así que tu código no cambia.
- **Respeta los tiempos de vida de la DI** — inyectar un servicio `Scoped` (vive una petición) dentro de uno `Singleton` (vive toda la app) "captura" el scoped y lo congela: es la *captive dependency*, fuente de bugs sutiles con datos obsoletos. `WebApplication.CreateBuilder` valida esto en desarrollo si no lo desactivas.
- **El comportamiento debe depender del entorno** — ASP.NET Core sabe si corre en `Development` o `Production` (`ASPNETCORE_ENVIRONMENT`). La página de excepciones con detalles va solo en desarrollo; en producción filtraría información interna a cualquiera.

## Recursos didácticos divertidos

El propio SDK te monta un proyecto de ejemplo funcionando en segundos con `dotnet new webapi` y `dotnet run`: la forma más rápida de ver un `Program.cs` real y toquetearlo. Para los códigos de estado HTTP que devolverás sin parar, <https://http.cat/> los explica con gatos.

---

*En resumen: ASP.NET Core es el mismo .NET que ya conoces, pero convertido en un servidor que nunca termina — te da el servidor web (Kestrel), la DI, la configuración, el logging, el middleware y el routing ya integrados para que solo escribas tu lógica.*
