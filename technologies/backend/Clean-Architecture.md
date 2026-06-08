# Clean Architecture

## ¿Qué es?

Clean Architecture es un patrón de organización del código que separa la lógica de negocio de los detalles de infraestructura (base de datos, HTTP, frameworks) mediante capas con dependencias que solo apuntan hacia adentro.

## ¿Por qué existe?

El problema clásico: con el tiempo, la lógica de negocio queda mezclada con EF Core, controladores ASP.NET o servicios externos. Cambiar la base de datos o el ORM se vuelve una refactorización masiva.

Si vienes de **N-Layer Architecture** (la típica estructura Presentación / Negocio / Datos de proyectos .NET clásicos), la diferencia clave es la dirección de las dependencias. En N-Layer, el negocio depende de la capa de datos. En Clean Architecture, es al revés: la infraestructura depende del dominio, nunca al contrario. El dominio no sabe que existe EF Core.

## ¿Cómo encaja en este proyecto?

EcoWaveProjectManagement organiza cada módulo (Projectes, Imatges, Dissenys) siguiendo esta estructura de capas:

```
/src
  /Domain          → Entidades, interfaces de repositorio, reglas de negocio
  /Application     → Casos de uso, DTOs, interfaces de servicios
  /Infrastructure  → EF Core, repositorios concretos, servicios externos
  /API             → Controladores ASP.NET Core, configuración DI
```

Las dependencias fluyen así: `API → Application → Domain` e `Infrastructure → Domain`. Infrastructure implementa las interfaces que Domain define, nunca al revés.

## Lo mínimo que necesitas saber

**1. El dominio no tiene dependencias externas**

```csharp
// Domain/Entities/Projecte.cs
public class Projecte
{
    public Guid Id { get; private set; }
    public string Nom { get; private set; }
    // Sin using de EF Core ni nada externo
}
```

**2. Las interfaces viven en Domain/Application, las implementaciones en Infrastructure**

```csharp
// Application/Interfaces/IProjecteRepository.cs
public interface IProjecteRepository
{
    Task<Projecte?> GetByIdAsync(Guid id);
}

// Infrastructure/Repositories/ProjecteRepository.cs
public class ProjecteRepository : IProjecteRepository
{
    private readonly AppDbContext _db;
    // EF Core solo aquí
}
```

**3. Los casos de uso orquestan la lógica**

```csharp
// Application/UseCases/GetProjecteUseCase.cs
public class GetProjecteUseCase(IProjecteRepository repo)
{
    public Task<Projecte?> ExecuteAsync(Guid id) => repo.GetByIdAsync(id);
}
```

**4. La inyección de dependencias une todo en la capa API**

```csharp
// API/Program.cs
builder.Services.AddScoped<IProjecteRepository, ProjecteRepository>();
builder.Services.AddScoped<GetProjecteUseCase>();
```

## Lo que NO hace

- No es una estructura de carpetas obligatoria — es un principio de dependencias.
- No elimina la necesidad de EF Core ni de ASP.NET; los aleja del núcleo.
- No es lo mismo que CQRS ni que DDD, aunque se combinan con frecuencia.
- No protege sola contra código mal escrito — requiere disciplina en cada PR.

---

*En resumen: Clean Architecture garantiza que el corazón del proyecto (las reglas de negocio de Projectes, Imatges y Dissenys) pueda entenderse, probarse y modificarse sin tocar ningún framework ni base de datos.*
