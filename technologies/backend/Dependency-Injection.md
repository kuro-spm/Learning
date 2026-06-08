# Dependency Injection

## ¿Qué es?

Dependency Injection (DI) es un patrón en el que las dependencias de una clase se proporcionan desde fuera en lugar de que la propia clase las cree. En .NET, el contenedor de DI integrado gestiona la creación, el ciclo de vida y la resolución de esas dependencias automáticamente.

## ¿Por qué existe?

Sin DI, cada clase instancia sus propias dependencias con `new`, lo que genera acoplamiento fuerte, dificulta los tests y complica el mantenimiento.

Si vienes de Unity o de versiones antiguas de .NET Framework, quizá conoces **Service Locator** (como `ServiceLocator.GetInstance<T>()`). La diferencia clave es que con DI el contenedor *empuja* las dependencias hacia la clase, mientras que con Service Locator la clase las *pide* activamente — lo que sigue ocultando dependencias y dificulta el testing.

## ¿Cómo encaja en este proyecto?

EcoWaveProjectManagement registra todos sus servicios en `Program.cs` usando el contenedor integrado de .NET 10. Cada módulo (Projectes, Imatges, Dissenys) expone sus servicios de dominio e infraestructura, que se registran aquí y se inyectan automáticamente en los controllers y handlers correspondientes.

```csharp
// Program.cs
builder.Services.AddScoped<IProjectesService, ProjectesService>();
builder.Services.AddScoped<IImatgesRepository, ImatgesRepository>();
builder.Services.AddSingleton<IStorageClient, AzureBlobStorageClient>();
```

## Lo mínimo que necesitas saber

**1. Los tres ciclos de vida**

| Ciclo | Método | Cuándo usarlo |
|---|---|---|
| Transient | `AddTransient` | Servicios sin estado, creados cada vez |
| Scoped | `AddScoped` | Una instancia por request HTTP |
| Singleton | `AddSingleton` | Instancias compartidas durante toda la app |

**2. Inyección por constructor** — la forma recomendada en este proyecto:

```csharp
public class ProjectesService : IProjectesService
{
    private readonly IProjectesRepository _repo;

    public ProjectesService(IProjectesRepository repo)
    {
        _repo = repo;
    }
}
```

**3. Interfaces como contratos** — siempre registra contra la interfaz, no la implementación concreta. Así puedes sustituir implementaciones sin tocar el código que las consume.

**4. Registro por módulo** — para mantener `Program.cs` limpio, cada módulo puede exponer un método de extensión:

```csharp
// Dissenys/DissenysDependencies.cs
public static class DissenysDependencies
{
    public static IServiceCollection AddDissenysServices(this IServiceCollection services)
    {
        services.AddScoped<IDissenysService, DissenysService>();
        return services;
    }
}
```

```csharp
// Program.cs
builder.Services.AddDissenysServices();
```

## Lo que NO hace

- **No es un framework de arquitectura** — DI no impone capas ni patrones como CQRS o Repository. Eso lo decide el equipo.
- **No resuelve dependencias circulares** — si el servicio A depende de B y B depende de A, el contenedor lanzará una excepción en runtime.
- **No gestiona configuración** — para leer `appsettings.json` se usa `IOptions<T>`, que sí se integra con DI pero es un sistema separado.

---

*En resumen: el contenedor de DI de .NET es el pegamento invisible que conecta los módulos de EcoWaveProjectManagement — registras una vez en `Program.cs` y el framework se encarga de construir y entregar cada dependencia donde se necesita.*
