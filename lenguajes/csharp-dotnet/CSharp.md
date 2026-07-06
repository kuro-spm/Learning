# C#

## ¿Qué es?

C# es un lenguaje de programación orientado a objetos, fuertemente tipado y compilado, que corre sobre el runtime de .NET. Es habitual usarlo como lenguaje principal del backend en aplicaciones web y APIs.

## ¿Por qué existe?

C# nació para ofrecer productividad de lenguaje de alto nivel con rendimiento cercano al nativo. Si vienes de VB.NET, la diferencia principal es que C# tiene un ecosistema moderno mucho más activo, sintaxis más expresiva y es el lenguaje de referencia para todo el tooling de .NET (ASP.NET Core, Entity Framework, etc.). Si vienes de Java, C# añade características como `record`, `pattern matching`, `nullable reference types` y `async/await` con una integración más profunda en el lenguaje.

## ¿Cuándo y para qué se usa?

C# se usa como lenguaje de backend en cualquier aplicación que exponga una API REST: una tienda online que gestiona productos y pedidos, un sistema de facturación, o una app de gestión de tareas. Los endpoints se implementan como controladores o Minimal APIs en ASP.NET Core, mientras que un frontend (React, Angular o cualquier otro) consume esa API. El acceso a base de datos, la lógica de negocio y la validación viven en C#.

## Lo mínimo que necesitas saber

**1. Records para DTOs**
Los objetos de transferencia de datos entre capas se modelan con `record` para inmutabilidad y comparación por valor automática.

```csharp
public record ProductoDto(int Id, string Nombre, DateTime FechaCreacion);
```

**2. Minimal APIs o Controllers**
Puedes organizar los endpoints con Controllers de ASP.NET Core (uno por recurso o módulo) o con Minimal APIs para casos más ligeros.

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductosController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductoDto>> GetById(int id) { ... }
}
```

**3. async/await en toda la capa de acceso a datos**
Todas las operaciones de I/O (base de datos, ficheros) son asíncronas para no bloquear el hilo del servidor.

```csharp
public async Task<Producto?> GetByIdAsync(int id) =>
    await _context.Productos.FindAsync(id);
```

**4. Nullable reference types activado**
Es habitual tener `<Nullable>enable</Nullable>` en el `.csproj`. Esto significa que el compilador avisa si puedes recibir `null` sin gestionarlo.

```xml
<Nullable>enable</Nullable>
```

**5. Dependency Injection nativa**
Los servicios y repositorios se registran en `Program.cs` y se inyectan por constructor sin librerías externas.

```csharp
builder.Services.AddScoped<IProductoService, ProductoService>();
```

## Lo que NO hace

- C# por sí solo no gestiona rutas HTTP ni serialización JSON — eso lo hace ASP.NET Core.
- No es un framework: es el lenguaje. El framework que lo rodea es .NET 10 / ASP.NET Core.
- No compila a JavaScript ni corre en el frontend — el frontend React es completamente independiente.

---

*En resumen: C# con .NET es la columna vertebral de un backend moderno; cada área de la aplicación es un conjunto de clases C# que define la lógica, los modelos y los endpoints que el frontend consume.*
