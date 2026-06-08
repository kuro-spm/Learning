# xUnit

## ¿Qué es?

xUnit es el framework de tests unitarios estándar para proyectos .NET modernos. Permite escribir, organizar y ejecutar pruebas automatizadas directamente desde la CLI o desde el IDE.

## ¿Por qué existe?

NUnit y MSTest resolvían el problema de los tests en .NET durante años, pero venían con convenciones heredadas (atributos `[SetUp]`, estado compartido entre tests) que favorecían código de test frágil y difícil de mantener. xUnit nació para corregirlo: cada test corre en una instancia nueva de la clase, el constructor reemplaza al `[SetUp]`, y se elimina el estado global por diseño. Es el framework que Microsoft recomienda para proyectos .NET actuales.

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, xUnit es el framework base para los tests unitarios del backend (.NET 10). Cada módulo (`Projectes`, `Imatges`, `Dissenys`) tiene su propio proyecto de test que referencia el proyecto de aplicación correspondiente. Los tests validan la lógica de negocio de servicios y repositorios sin levantar la aplicación completa.

## Lo mínimo que necesitas saber

**1. Un test es un método con `[Fact]`**
```csharp
public class ProjecteServiceTests
{
    [Fact]
    public void CrearProjecte_NomBuit_LlancaExcepcio()
    {
        var service = new ProjecteService();
        Assert.Throws<ArgumentException>(() => service.Crear(""));
    }
}
```

**2. Tests parametrizados con `[Theory]` y `[InlineData]`**
```csharp
[Theory]
[InlineData("")]
[InlineData(null)]
[InlineData("   ")]
public void CrearProjecte_NomInvalid_LlancaExcepcio(string nom)
{
    var service = new ProjecteService();
    Assert.Throws<ArgumentException>(() => service.Crear(nom));
}
```

**3. El constructor es el setup**
```csharp
public class DissenysServiceTests
{
    private readonly DissenysService _service;

    public DissenysServiceTests()
    {
        _service = new DissenysService(new FakeRepository());
    }
}
```

**4. Ejecutar tests desde la CLI**
```bash
dotnet test
dotnet test --filter "FullyQualifiedName~ProjecteService"
```

**5. Proyecto de test en el `.csproj`**
```xml
<PackageReference Include="xunit" Version="2.9.3" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
```

## Lo que NO hace

- No hace mocking por si solo — para eso se usa una librería separada como **Moq** o **NSubstitute**.
- No levanta el servidor HTTP ni la base de datos — eso es territorio de tests de integración con `WebApplicationFactory`.
- No genera reportes HTML automáticamente — requiere herramientas adicionales como `dotnet-reportgenerator`.

---

*En resumen: xUnit es el punto de partida de cualquier test en el backend de EcoWaveProjectManagement — simple, sin estado global y alineado con las convenciones modernas de .NET.*
