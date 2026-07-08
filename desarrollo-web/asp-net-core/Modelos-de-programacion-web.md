# Modelos de programación web de ASP.NET Core

## ¿Qué es?

ASP.NET Core no es un único estilo de programar: sobre su mismo motor (servidor, DI, middleware, routing) ofrece **varios modelos de programación web**, que son formas distintas de definir cómo entra una petición y qué código la atiende. Los cinco que verás son **MVC**, **Razor Pages**, **Web API**, **Minimal APIs** y **Blazor**. MVC es solo uno de ellos, no "la forma" de hacer ASP.NET.

## ¿Por qué existe?

No todas las aplicaciones web tienen la misma forma: unas devuelven páginas HTML completas al navegador, otras exponen datos (JSON) para una app móvil o un frontend de React, y otras son interfaces interactivas que viven en el propio navegador. Un solo estilo obligaría a forzar la misma herramienta para problemas muy distintos.

Por eso ASP.NET Core separa el **motor común** (lo que explica [De .NET Core a ASP.NET Core](De-NET-Core-a-ASP-NET-Core.md)) de los **modelos de programación** que se montan encima. Eliges el que encaja con lo que construyes, y todos comparten la misma DI, la misma configuración y el mismo pipeline.

> Si la sensación al leer sobre Data Annotations o controladores era "esto es MVC y ASP.NET va siempre de MVC", el matiz es este: MVC es un patrón de presentación popular y veterano, y por eso los tutoriales suelen empezar por ahí, pero el framework no te ata a él.

## ¿Cuándo y para qué se usa?

Cada modelo brilla en un escenario distinto. Piensa en el tipo de aplicación que estás construyendo:

- **Un backend que sirve datos** a un frontend de React/Angular o a una app móvil → **Web API** o **Minimal APIs**.
- **Un sitio que genera HTML en el servidor** (un blog, un panel de administración clásico) → **Razor Pages** (o **MVC** si la app es grande y con mucha lógica de navegación).
- **Una interfaz rica e interactiva escrita en C#** en lugar de JavaScript → **Blazor**.

## Lo mínimo que necesitas saber

Los cinco modelos, con el gesto mínimo de cada uno para atender "dame los productos":

**1. MVC — controlador + vista**

El patrón Modelo-Vista-Controlador clásico: un controlador prepara los datos y elige una **vista** que renderiza HTML. Pensado para sitios grandes con muchas pantallas.

```csharp
public class ProductsController : Controller
{
    public IActionResult Index() => View(_repo.GetAll());  // devuelve una página HTML
}
```

**2. Razor Pages — una página, su lógica al lado**

Cada página web tiene su archivo `.cshtml` (el HTML) y su `.cshtml.cs` (la lógica). Más directo que MVC para CRUD y formularios, sin repartir todo entre controladores.

```csharp
// Products.cshtml.cs
public class ProductsModel : PageModel
{
    public List<Product> Products { get; set; } = [];
    public void OnGet() => Products = _repo.GetAll();
}
```

**3. Web API — controladores que devuelven datos**

Como MVC, pero el controlador hereda de `ControllerBase` (sin vistas) y lo que devuelve se serializa a **JSON**. Es lo que usas para una API REST. Aquí es donde aparecen `[ApiController]`, el [model binding](Model-Binding.md) y la [validación con Data Annotations](Validacion-con-DataAnnotations.md).

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IEnumerable<Product> Get() => _repo.GetAll();  // → JSON
}
```

**4. Minimal APIs — endpoints sin controladores**

Defines la ruta y su código directamente en `Program.cs`, sin clase de controlador. Muy ligero para APIs pequeñas o microservicios.

```csharp
app.MapGet("/api/products", (IProductRepository repo) => repo.GetAll());
```

**5. Blazor — UI en C#, con componentes**

Construyes la interfaz con componentes (como en React o Angular) pero escribiendo C# en lugar de JavaScript. El navegador ejecuta tu código (vía WebAssembly) o lo sincroniza con el servidor.

```razor
@page "/products"
@foreach (var p in products) { <li>@p.Name</li> }
```

**6. El modelo de programación es la capa de presentación, no la arquitectura de tu app**

Aquí está la respuesta a "¿ASP.NET acepta Clean Architecture?": **sí, y no compiten**. Estos modelos deciden *cómo entra la petición HTTP*; una arquitectura como [Clean Architecture](../../arquitectura-de-software/clean-architecture/Clean-Architecture.md) decide *cómo organizas las capas de tu aplicación* (dominio, casos de uso, infraestructura). ASP.NET Core ocupa solo la capa de **presentación**: tu controlador o tu endpoint recibe la petición y delega en un caso de uso que no sabe nada de HTTP. Es una de las combinaciones más habituales en proyectos .NET serios.

**7. Cuidado con acoplar el dominio al framework: Data Annotations vs FluentValidation**

Las [Data Annotations](Validacion-con-DataAnnotations.md) (`[Required]`, `[Range]`...) son atributos de una librería de .NET. Colocarlas sobre las **entidades del dominio** ata tu lógica de negocio a un detalle de presentación, justo lo que Clean Architecture intenta evitar. La pauta habitual por capas:

- **Data Annotations** en los **DTOs/modelos de entrada** de la capa de presentación (validación de formato: "el email tiene forma de email").
- **[FluentValidation](../de-wpf-a-web/FluentValidation.md)** en la capa de **aplicación/dominio** para reglas de negocio ("el email no está ya registrado"), en código y desacoplada de atributos.

## Lo que NO hace

- **No te obliga a elegir uno solo** — una misma app puede tener Razor Pages para el panel de administración y Web API para su API pública, sobre el mismo `Program.cs`.
- **No define la arquitectura de tu aplicación** — ninguno de estos modelos te dice cómo separar dominio, casos de uso e infraestructura; eso lo decides tú.
- **MVC no es obligatorio** — puedes construir una API completa sin escribir jamás un controlador MVC, solo con Minimal APIs.
- **Blazor no es "MVC en el navegador"** — es un modelo de componentes con estado, más parecido a React que a MVC.

## Buenas prácticas avanzadas

- **Elige por la forma de la salida, no por costumbre** — la pregunta que decide el modelo es "¿qué devuelvo?": HTML renderizado en servidor → Razor Pages/MVC; datos para otro cliente → Web API/Minimal APIs; UI interactiva en C# → Blazor. Empezar con MVC "porque es lo que salía en el tutorial" para una API que solo sirve JSON te deja con vistas y `ViewData` que nunca usarás.
- **Mantén los controladores finos ("thin controllers")** — un controlador o endpoint debería recibir la petición, delegar en un servicio/caso de uso y devolver el resultado. Si empiezas a ver reglas de negocio o consultas a base de datos dentro del controlador, la lógica se está filtrando a la capa de presentación y perderás el aislamiento que da la arquitectura por capas.
- **Minimal APIs no es "para juguetes"** — desde .NET 7-8 soportan filtros, grupos de endpoints (`MapGroup`), inyección de dependencias y validación; para muchos servicios son una elección de producción perfectamente válida, no solo para prototipos.
- **Deja las entidades de dominio limpias de atributos del framework** — ni Data Annotations de validación ni atributos de EF Core sobre las clases del núcleo. Configura persistencia y validación *desde fuera* (Fluent API de EF Core, FluentValidation) para que el dominio siga sin depender de cómo se guarda ni de cómo entra por HTTP.

## Recursos didácticos

La documentación oficial tiene una comparativa directa, «Choose an ASP.NET Core web UI» y «APIs overview», útil para ver los modelos enfrentados: <https://learn.microsoft.com/aspnet/core/>. Y la forma más rápida de tocarlos es generarlos: `dotnet new webapi`, `dotnet new razor` y `dotnet new blazor` te montan un proyecto de cada tipo en segundos para comparar sus `Program.cs`.

---

*En resumen: ASP.NET Core te da varios modelos de programación web (MVC, Razor Pages, Web API, Minimal APIs, Blazor) sobre un mismo motor; MVC es solo uno y no es obligatorio, y ninguno impone la arquitectura de tu app — encajan sin problema como capa de presentación de una Clean Architecture.*
