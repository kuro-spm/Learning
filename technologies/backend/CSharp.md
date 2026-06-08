# C#

## ¿Qué es?

C# es el lenguaje principal del backend de EcoWaveProjectManagement. Es un lenguaje orientado a objetos, fuertemente tipado y compilado, que corre sobre el runtime de .NET.

## ¿Por qué existe?

C# nació para ofrecer productividad de lenguaje de alto nivel con rendimiento cercano al nativo. Si vienes de VB.NET, la diferencia principal es que C# tiene un ecosistema moderno mucho más activo, sintaxis más expresiva y es el lenguaje de referencia para todo el tooling de .NET (ASP.NET Core, Entity Framework, etc.). Si vienes de Java, C# añade características como `record`, `pattern matching`, `nullable reference types` y `async/await` con una integración más profunda en el lenguaje.

## ¿Cómo encaja en este proyecto?

EcoWaveProjectManagement usa C# con .NET 10 como única tecnología de backend. Los tres módulos del proyecto — **Projectes**, **Imatges** y **Dissenys** — están implementados como endpoints REST en ASP.NET Core. El frontend en React consume esa API. Todo el acceso a base de datos, la lógica de negocio y la validación viven en C#.

## Lo mínimo que necesitas saber

**1. Records para DTOs**
Los objetos de transferencia de datos entre capas se modelan con `record` para inmutabilidad y comparación por valor automática.

```csharp
public record ProjecteDto(int Id, string Nom, DateTime DataCreacio);
```

**2. Minimal APIs o Controllers**
El proyecto usa Controllers de ASP.NET Core para organizar los endpoints por módulo.

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProjectesController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<ActionResult<ProjecteDto>> GetById(int id) { ... }
}
```

**3. async/await en toda la capa de acceso a datos**
Todas las operaciones de I/O (base de datos, ficheros de imágenes) son asíncronas para no bloquear el hilo del servidor.

```csharp
public async Task<Projecte?> GetByIdAsync(int id) =>
    await _context.Projectes.FindAsync(id);
```

**4. Nullable reference types activado**
El proyecto tiene `<Nullable>enable</Nullable>` en el `.csproj`. Esto significa que el compilador avisa si puedes recibir `null` sin gestionarlo.

```xml
<Nullable>enable</Nullable>
```

**5. Dependency Injection nativa**
Los servicios y repositorios se registran en `Program.cs` y se inyectan por constructor sin librerías externas.

```csharp
builder.Services.AddScoped<IProjecteService, ProjecteService>();
```

## Lo que NO hace

- C# por sí solo no gestiona rutas HTTP ni serialización JSON — eso lo hace ASP.NET Core.
- No es un framework: es el lenguaje. El framework que lo rodea es .NET 10 / ASP.NET Core.
- No compila a JavaScript ni corre en el frontend — el frontend React es completamente independiente.

---

*En resumen: C# con .NET 10 es la columna vertebral de todo el backend; cada módulo del proyecto es un conjunto de clases C# que define la lógica, los modelos y los endpoints que el frontend React consume.*
