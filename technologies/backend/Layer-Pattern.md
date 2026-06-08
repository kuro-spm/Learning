# Layer Pattern

## ¿Qué es?

Es una forma de organizar el código en capas con responsabilidades bien definidas: **Domain**, **Application** e **Infrastructure**. Cada capa solo puede depender de la que tiene por debajo, nunca al revés.

## ¿Por qué existe?

Sin capas, la lógica de negocio termina mezclada con llamadas a base de datos, validaciones, HTTP y todo lo demás — el clásico "Big Ball of Mud". Si ya conoces la arquitectura en N-capas de ASP.NET clásico (Presentation / Business Logic / Data Access), el Layer Pattern es la misma idea pero con límites más estrictos y orientado a módulos en lugar de a la aplicación entera. La diferencia clave: aquí las capas **no se comparten entre módulos**; cada módulo tiene las suyas propias.

## ¿Cómo encaja en este proyecto?

EcoWaveProjectManagement organiza sus módulos (Projectes, Imatges, Dissenys) de esta forma:

```
Modules/
  Projectes/
    Domain/          ← entidades, interfaces de repositorio, reglas de negocio
    Application/     ← casos de uso, DTOs, servicios de aplicación
    Infrastructure/  ← EF Core, repositorios concretos, integraciones externas
```

La API de ASP.NET Core actúa como punto de entrada y delega en la capa **Application**. La capa **Infrastructure** implementa los contratos definidos en **Domain** (inversión de dependencias).

## Lo mínimo que necesitas saber

**1. Domain no depende de nada externo**

```csharp
// Domain/Entities/Projecte.cs
public class Projecte
{
    public Guid Id { get; private set; }
    public string Nom { get; private set; }

    public void Renombrar(string nouNom)
    {
        if (string.IsNullOrWhiteSpace(nouNom)) throw new ArgumentException("Nom invàlid");
        Nom = nouNom;
    }
}
```

**2. Application orquesta, no implementa**

```csharp
// Application/UseCases/CrearProjecteUseCase.cs
public class CrearProjecteUseCase(IProjecteRepository repo)
{
    public async Task<Guid> ExecuteAsync(CrearProjecteDto dto)
    {
        var projecte = new Projecte(dto.Nom);
        await repo.AfegirAsync(projecte);
        return projecte.Id;
    }
}
```

**3. Infrastructure implementa las interfaces del Domain**

```csharp
// Infrastructure/Repositories/ProjecteRepository.cs
public class ProjecteRepository(AppDbContext db) : IProjecteRepository
{
    public async Task AfegirAsync(Projecte p) => await db.Projectes.AddAsync(p);
}
```

**4. El registro de dependencias conecta todo**

```csharp
// Program.cs o módulo de extensión
builder.Services.AddScoped<IProjecteRepository, ProjecteRepository>();
builder.Services.AddScoped<CrearProjecteUseCase>();
```

## Lo que NO hace

- **No es Clean Architecture ni DDD** — los toma como inspiración, pero es una estructura más ligera y pragmática.
- **No impide errores de diseño** — puedes meter lógica de negocio en Infrastructure y el compilador no se quejará; la disciplina es del equipo.
- **No gestiona la comunicación entre módulos** — eso es responsabilidad de otro patrón (eventos de dominio, mediator, etc.).

---

*En resumen: el Layer Pattern divide cada módulo en Domain, Application e Infrastructure para que la lógica de negocio nunca dependa de detalles técnicos, facilitando el mantenimiento y los tests.*
