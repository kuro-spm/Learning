# Module-based Organization

## ¿Qué es?

Es una estrategia de estructura de carpetas y responsabilidades en la que el código backend se agrupa por **dominio funcional** (módulos) en lugar de por tipo técnico. Cada módulo contiene todo lo que necesita para funcionar: controladores, servicios, modelos y repositorios propios.

## ¿Por qué existe?

La organización tradicional en .NET tiende a agrupar por capa técnica: una carpeta `Controllers/`, otra `Services/`, otra `Models/`. Esto funciona bien en proyectos pequeños, pero escala mal: cuando el proyecto crece, modificar una funcionalidad obliga a saltar entre múltiples carpetas sin relación visual.

La organización por módulos agrupa todo lo relacionado con una funcionalidad en un solo lugar. Si ya conoces la arquitectura **Vertical Slice** o el patrón **Feature Folders**, esto es exactamente eso aplicado a escala de módulo, sin la rigidez de un framework externo.

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement** el backend está dividido en tres módulos funcionales:

| Módulo | Responsabilidad |
|---|---|
| `Projectes` | Gestión del ciclo de vida de proyectos |
| `Imatges` | Subida, almacenamiento y consulta de imágenes |
| `Dissenys` | Creación y vinculación de diseños a proyectos |

La estructura de carpetas refleja esta división directamente:

```
src/
  Modules/
    Projectes/
      ProjectesController.cs
      ProjectesService.cs
      ProjectesRepository.cs
      ProjecteDto.cs
    Imatges/
      ImatgesController.cs
      ImatgesService.cs
      ImatgeDto.cs
    Dissenys/
      DissenysController.cs
      DissenysService.cs
      DissenyDto.cs
```

## Lo mínimo que necesitas saber

**1. Cada módulo es autónomo.** Sus clases internas son `internal` por defecto; solo expone lo necesario al exterior.

```csharp
// Projectes/ProjectesService.cs
internal class ProjectesService
{
    public async Task<ProjecteDto> GetByIdAsync(int id) { ... }
}
```

**2. El registro de dependencias puede hacerse por módulo.** Cada módulo tiene su propio método de extensión para `IServiceCollection`:

```csharp
// Projectes/ProjectesModule.cs
public static class ProjectesModule
{
    public static IServiceCollection AddProjectes(this IServiceCollection services)
    {
        services.AddScoped<ProjectesService>();
        services.AddScoped<ProjectesRepository>();
        return services;
    }
}

// Program.cs
builder.Services.AddProjectes();
builder.Services.AddImatges();
builder.Services.AddDissenys();
```

**3. La comunicación entre módulos va por interfaces, no por referencia directa.** Si `Dissenys` necesita datos de `Projectes`, lo hace a través de una interfaz pública, nunca instanciando clases internas de otro módulo.

**4. Los DTOs son locales al módulo.** `ProjecteDto` vive en `Projectes/`, no en una carpeta `Shared/Models/` global.

## Lo que NO hace

- No es un microservicio: todo sigue siendo un único proceso .NET desplegado junto.
- No impone ningún framework externo — es una convención de carpetas y namespaces.
- No reemplaza la inyección de dependencias estándar de .NET; la usa.
- No garantiza aislamiento de base de datos: todos los módulos pueden compartir el mismo `DbContext` si el proyecto lo decide.

---

*En resumen: Module-based Organization es simplemente acordar que cada carpeta de módulo es el hogar completo de una funcionalidad, lo que hace que EcoWaveProjectManagement sea más fácil de navegar, modificar y escalar sin necesidad de herramientas adicionales.*
