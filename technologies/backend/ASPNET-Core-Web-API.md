# ASP.NET Core Web API

## ¿Qué es?

ASP.NET Core Web API es el framework de Microsoft para construir servicios HTTP en .NET. Proporciona el pipeline de solicitud/respuesta, el enrutamiento y las herramientas de serialización que convierten tu código C# en una API REST accesible desde cualquier cliente HTTP.

## ¿Por qué existe?

Antes existía **ASP.NET Web API** (sobre .NET Framework), que dependía de IIS y no era multiplataforma. ASP.NET Core lo reimplementó desde cero para ser ligero, modular y ejecutable en Linux, macOS y Windows. La diferencia clave: en ASP.NET Core el servidor HTTP (Kestrel), la inyección de dependencias y el middleware son ciudadanos de primera clase, no capas opcionales añadidas encima.

## ¿Cómo encaja en este proyecto?

EcoWaveProjectManagement expone tres módulos — **Projectes**, **Imatges** y **Dissenys** — al frontend React mediante una API REST. ASP.NET Core Web API 10 es la capa que recibe cada petición HTTP entrante, la enruta al controlador correcto, invoca la lógica de negocio y serializa la respuesta JSON que React consume.

## Lo mínimo que necesitas saber

**1. Controladores y atributos de ruta**

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProjectesController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<ActionResult<ProjecteDto>> GetById(int id) { ... }

    [HttpPost]
    public async Task<ActionResult<ProjecteDto>> Create(CreateProjecteRequest request) { ... }
}
```

**2. Inyección de dependencias integrada**

Los servicios se registran en `Program.cs` y se inyectan en el constructor del controlador:

```csharp
builder.Services.AddScoped<IProjecteService, ProjecteService>();
```

**3. Minimal APIs como alternativa**

Para endpoints simples puedes evitar los controladores:

```csharp
app.MapGet("/api/dissenys/{id}", async (int id, IDissenysService svc) =>
    await svc.GetByIdAsync(id) is { } d ? Results.Ok(d) : Results.NotFound());
```

**4. Middleware y pipeline**

El orden en `Program.cs` importa. La autenticación debe ir antes de la autorización:

```csharp
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

**5. Serialización automática**

`System.Text.Json` serializa y deserializa las respuestas automáticamente. Puedes configurar el comportamiento globalmente:

```csharp
builder.Services.ConfigureHttpJsonOptions(opts =>
    opts.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase);
```

## Lo que NO hace

- **No es un ORM**: no accede a la base de datos directamente; eso lo hace Entity Framework Core u otro repositorio.
- **No gestiona la lógica de negocio**: los controladores deben delegar en servicios, no contener reglas de dominio.
- **No sirve el frontend React**: ese trabajo lo hace un servidor de estáticos separado (Vite, Nginx, etc.).

---

*En resumen: ASP.NET Core Web API es el punto de entrada HTTP de EcoWaveProjectManagement — enruta peticiones, valida modelos y devuelve JSON — mientras delega la lógica real a las capas de servicio y repositorio.*
