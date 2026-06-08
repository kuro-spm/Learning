# MSBuild

## ¿Qué es?

MSBuild (Microsoft Build Engine) es el sistema de build oficial de .NET. Es la herramienta que transforma tu código fuente C# en assemblies ejecutables o bibliotecas, gestionando dependencias, referencias y tareas de compilación.

## ¿Por qué existe?

Antes de MSBuild, compilar proyectos grandes en el ecosistema Microsoft requería scripts manuales o herramientas externas como NAnt. MSBuild estandarizó el proceso definiendo el build como un archivo XML declarativo (`.csproj`), permitiendo reproducibilidad y automatización.

Si vienes de usar `dotnet build` o el botón "Build" de Visual Studio, ya estás usando MSBuild sin saberlo — esos comandos lo invocan internamente. La diferencia es que MSBuild te da acceso directo a sus mecanismos: puedes personalizar targets, inyectar tareas y controlar el pipeline con precisión.

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement** cada módulo (Projectes, Imatges, Dissenys) tiene su propio `.csproj`. MSBuild es responsable de:

- Compilar cada módulo de forma independiente o conjunta desde la solución `.sln`
- Gestionar referencias entre proyectos (por ejemplo, si Imatges depende de un proyecto de infraestructura compartido)
- Ejecutar tareas pre/post-build, como copiar archivos estáticos o lanzar migraciones de EF Core en CI/CD

```bash
# Compilar toda la solución
dotnet build EcoWaveProjectManagement.sln --configuration Release

# Compilar solo el módulo de Projectes
dotnet build src/Projectes/Projectes.csproj
```

## Lo mínimo que necesitas saber

**1. El archivo `.csproj` es el script de MSBuild**

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\Shared\Shared.csproj" />
  </ItemGroup>
</Project>
```

**2. Targets y Tasks**

Un *target* es un grupo de pasos nombrado. Una *task* es una acción concreta dentro de un target (copiar archivos, ejecutar un script, etc.).

```xml
<Target Name="CopyAssets" AfterTargets="Build">
  <Copy SourceFiles="assets\logo.png" DestinationFolder="$(OutputPath)" />
</Target>
```

**3. Propiedades y Items**

Las *propiedades* son variables escalares (`$(OutputPath)`). Los *items* son colecciones de ficheros (`@(Compile)`). Distinguir ambos evita errores comunes al personalizar builds.

**4. Configuraciones**

`Debug` y `Release` son configuraciones predefinidas que alteran optimizaciones y símbolos. Puedes añadir configuraciones propias para entornos específicos del proyecto.

**5. MSBuild en CI/CD**

En el pipeline del proyecto, `dotnet publish` invoca MSBuild para generar el artefacto desplegable. Entender qué target ejecuta cada comando ayuda a depurar fallos de pipeline.

## Lo que NO hace

- No gestiona paquetes NuGet directamente — eso lo hace `dotnet restore` (que también llama a MSBuild internamente, pero la resolución la hace NuGet).
- No es un task runner de propósito general como Make o Gradle; está optimizado para el ecosistema .NET.
- No reemplaza a Docker ni a los scripts de CI — MSBuild produce el binario, el resto del pipeline lo empaqueta y despliega.

---

*En resumen: MSBuild es el motor invisible que convierte tu código C# en software ejecutable; los archivos `.csproj` son su configuración, y entenderlos te da control real sobre cómo se construye y publica cada módulo de EcoWaveProjectManagement.*
