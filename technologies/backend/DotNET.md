# .NET 10

## ¿Qué es?

.NET 10 es el runtime y plataforma de ejecución unificada de Microsoft para aplicaciones C#. Es la versión LTS (Long-Term Support) actual, lo que significa soporte garantizado durante tres años.

## ¿Por qué existe?

.NET Framework (el clásico de Windows) era lento de evolucionar, dependía del sistema operativo y no corría en Linux ni macOS. .NET Core surgió para resolver eso, y .NET 5 en adelante unificó ambos mundos bajo un único nombre: `.NET`. Cada versión par con LTS (6, 8, 10) es la que se usa en producción. Si vienes de .NET Framework, la diferencia principal es que aquí todo es multiplataforma, más rápido y con un ciclo de releases predecible.

## ¿Cómo encaja en este proyecto?

EcoWaveProjectManagement usa .NET 10 como base de toda la API backend. Los tres módulos principales —**Projectes**, **Imatges** y **Dissenys**— son proyectos C# que compilan y corren sobre este runtime. El servidor HTTP, la serialización JSON, la inyección de dependencias y la capa de acceso a datos dependen directamente de las APIs nativas de .NET 10.

## Lo mínimo que necesitas saber

**1. Minimal APIs (el estilo que usa este proyecto)**

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/projectes", () => Results.Ok(new[] { "Eco-001", "Eco-002" }));

app.Run();
```

**2. Inyección de dependencias integrada**

```csharp
builder.Services.AddScoped<IProjecteService, ProjecteService>();
```

No necesitas contenedores externos como Autofac; el DI container de .NET es suficiente para este proyecto.

**3. Configuración por capas**

```csharp
// appsettings.json -> appsettings.Development.json -> variables de entorno
var connString = builder.Configuration.GetConnectionString("DefaultConnection");
```

**4. Publicación y despliegue**

```bash
dotnet publish -c Release -o ./publish
```

Genera un artefacto autocontenido listo para correr en cualquier entorno con .NET 10 instalado.

**5. Target framework en el `.csproj`**

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
</PropertyGroup>
```

## Lo que NO hace

- No es un framework web por sí solo — para las rutas HTTP se usa **ASP.NET Core**, que corre encima de .NET.
- No gestiona la base de datos — eso es responsabilidad de **Entity Framework Core** u otro ORM.
- No sustituye a Docker ni a ningún orquestador; solo provee el runtime de ejecución.

---

*En resumen: .NET 10 es el motor que hace correr todo el backend de EcoWaveProjectManagement. No interactuas con él directamente en el día a día, pero entender su ciclo de vida, su sistema de configuración y su DI container te ahorrará muchas horas de depuración.*
