# xunit.runner.visualstudio

## ¿Qué es?

Es el adaptador que conecta el framework de testing xUnit con Visual Studio y la CLI de .NET. Sin él, `dotnet test` y el Test Explorer de Visual Studio no pueden descubrir ni ejecutar tests escritos con xUnit.

## ¿Por qué existe?

.NET no tiene un runner de tests universal integrado: cada framework (xUnit, NUnit, MSTest) necesita un adaptador que traduzca sus tests al protocolo VSTest. Si vienes de MSTest, ese adaptador ya viene incluido por defecto. Con xUnit hay que añadirlo explícitamente. Sin este paquete, ejecutar `dotnet test` simplemente no encontraría ningún test.

## ¿Cómo encaja en este proyecto?

En EcoWaveProjectManagement, los tests unitarios e de integración de los módulos **Projectes**, **Imatges** y **Dissenys** se escriben con xUnit. Este adaptador es lo que permite que el pipeline de CI/CD y cualquier desarrolladora en local ejecute la suite completa con un solo comando:

```bash
dotnet test
```

También es el que hace que el Test Explorer de Visual Studio muestre los tests organizados por módulo y permita ejecutarlos o depurarlos de forma individual.

La referencia en cada proyecto de tests es:

```xml
<PackageReference Include="xunit.runner.visualstudio" Version="3.1.4">
  <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  <PrivateAssets>all</PrivateAssets>
</PackageReference>
```

`PrivateAssets="all"` asegura que este paquete no se propague como dependencia transitiva al resto de proyectos.

## Lo mínimo que necesitas saber

**1. No escribes código contra este paquete.** Es infraestructura pura — se referencia pero no se usa en el código de los tests.

**2. Requiere `Microsoft.NET.Test.Sdk` para funcionar.**

```xml
<PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.x.x" />
<PackageReference Include="xunit" Version="2.x.x" />
<PackageReference Include="xunit.runner.visualstudio" Version="3.1.4" />
```

**3. Habilita el filtrado por categoría desde la CLI:**

```bash
dotnet test --filter "Category=Projectes"
```

**4. Permite ver resultados detallados en CI:**

```bash
dotnet test --logger "trx;LogFileName=resultats.trx"
```

**5. Compatible con cobertura de código** cuando se combina con `coverlet.collector`.

## Lo que NO hace

- No es el framework de assertions — eso lo hace `xUnit` (o `FluentAssertions`).
- No genera informes HTML por sí solo — necesitas herramientas adicionales como ReportGenerator.
- No ejecuta tests en paralelo por sí mismo — la paralelización la gestiona xUnit internamente.
- No afecta al código de producción; sus ensamblados no se incluyen en el build final.

---

*En resumen: `xunit.runner.visualstudio` es el puente silencioso que hace que tus tests de xUnit sean visibles y ejecutables tanto desde el terminal como desde el IDE — sin él, los tests existen pero nadie los ve.*
