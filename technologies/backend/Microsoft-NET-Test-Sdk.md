# Microsoft.NET.Test.Sdk

## ¿Qué es?

Es el SDK base que necesita el CLI de .NET para descubrir, cargar y ejecutar tests en cualquier proyecto de pruebas. Sin él, `dotnet test` no sabe que el proyecto contiene tests.

## ¿Por qué existe?

Antes de que existiera este SDK unificado, cada framework de testing (NUnit, MSTest, xUnit) tenía su propio runner y su propia forma de integrarse con Visual Studio o la línea de comandos. Esto generaba inconsistencias: un test podía ejecutarse en el IDE pero no en CI, o viceversa.

`Microsoft.NET.Test.Sdk` centraliza la infraestructura: define los targets de MSBuild necesarios, configura el protocolo VSTest y actúa como punto de entrada común para cualquier adaptador de testing (xUnit, NUnit, MSTest). Si vienes del mundo de NUnit "standalone", la diferencia clave es que este SDK no ejecuta los tests por sí solo — es la capa de andamiaje que conecta el runner con el adaptador del framework concreto.

## ¿Cómo encaja en este proyecto?

EcoWaveProjectManagement tiene módulos como **Projectes**, **Imatges** y **Dissenys**. Cada módulo con tests tiene su propio `.csproj` de tipo test, y todos incluyen esta referencia como prerequisito:

```xml
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.14.1" />
```

Esto permite ejecutar todos los tests del backend desde la raíz con un único comando:

```bash
dotnet test
```

O por módulo específico:

```bash
dotnet test backend/Projectes/Projectes.Tests/Projectes.Tests.csproj
```

## Lo mínimo que necesitas saber

**1. Siempre va acompañado de un adaptador.** Solo no hace nada. Necesita el adaptador del framework elegido:

```xml
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.14.1" />
<PackageReference Include="xunit" Version="2.9.3" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.9.3" />
```

**2. Habilita la propiedad `IsTestProject`.** Al incluirlo, MSBuild marca el proyecto como ejecutable por `dotnet test` — no necesitas configurar nada más.

**3. Controla el formato de salida.** Puedes cambiar el logger para CI:

```bash
dotnet test --logger "trx;LogFileName=resultados.trx"
```

**4. Compatible con `dotnet watch`.** Durante desarrollo puedes re-ejecutar tests en caliente:

```bash
dotnet watch test
```

**5. La versión importa.** La versión 17.x está alineada con .NET 8+ y .NET 10. No mezcles versiones antiguas con el SDK de .NET 10 que usa este proyecto.

## Lo que NO hace

- No es un framework de assertions — eso lo proveen xUnit, FluentAssertions, etc.
- No genera mocks — para eso existe Moq o NSubstitute.
- No sustituye a un framework de testing: sin xUnit o NUnit, no hay tests que ejecutar.
- No configura covertura de código — eso requiere `coverlet` u otra herramienta adicional.

---

*En resumen: `Microsoft.NET.Test.Sdk` es el puente silencioso entre `dotnet test` y el framework de testing que elijas; su presencia es obligatoria pero su configuración es mínima.*
