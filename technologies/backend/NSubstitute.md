# NSubstitute

## ¿Qué es?

NSubstitute es una librería de mocking para .NET que permite crear dobles de prueba (sustitutos) de interfaces y clases abstractas, controlando su comportamiento durante los tests unitarios.

## ¿Por qué existe?

Cuando una clase depende de servicios externos (base de datos, APIs, repositorios), los tests unitarios necesitan aislarla. Sin mocking, cada test arrastraría dependencias reales, haciéndolos lentos y frágiles.

Si conoces **Moq**, NSubstitute resuelve el mismo problema pero con una sintaxis más fluida y menos ceremonial. En Moq escribes `mock.Setup(x => x.Metodo()).Returns(valor)`; en NSubstitute es simplemente `substitute.Metodo().Returns(valor)`, sin distinción entre el objeto mock y el objeto configurado.

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement** los módulos Projectes, Imatges y Dissenys siguen una arquitectura en capas con repositorios e interfaces de servicio. NSubstitute se usa en los tests unitarios (xUnit) para sustituir dependencias como `IProjectRepository`, `IImageService` o `IDesignValidator`, permitiendo testear la lógica de negocio sin tocar la base de datos ni el sistema de ficheros.

## Lo mínimo que necesitas saber

**1. Crear un sustituto**

```csharp
var repo = Substitute.For<IProjectRepository>();
```

**2. Configurar valores de retorno**

```csharp
repo.GetByIdAsync(42).Returns(new Project { Id = 42, Name = "EcoWave" });
```

**3. Verificar que se llamó a un método**

```csharp
await repo.Received(1).SaveAsync(Arg.Any<Project>());
```

**4. Simular excepciones**

```csharp
repo.GetByIdAsync(-1).Throws(new NotFoundException("Proyecto no encontrado"));
```

**5. Uso completo en un test xUnit**

```csharp
[Fact]
public async Task GetProject_ReturnsProject_WhenExists()
{
    var repo = Substitute.For<IProjectRepository>();
    repo.GetByIdAsync(1).Returns(new Project { Id = 1 });

    var service = new ProjectService(repo);
    var result = await service.GetAsync(1);

    Assert.Equal(1, result.Id);
    await repo.Received(1).GetByIdAsync(1);
}
```

## Lo que NO hace

- No reemplaza a xUnit ni a FluentAssertions — NSubstitute solo gestiona los dobles de prueba, no las aserciones ni la estructura de los tests.
- No puede sustituir clases concretas no virtuales ni métodos estáticos.
- No valida automáticamente que los sustitutos se usaron; hay que llamar explícitamente a `Received()`.

---

*En resumen: NSubstitute te permite aislar la unidad de código que quieres testear sustituyendo sus dependencias por objetos controlados, con una sintaxis más limpia que Moq y perfectamente integrada en el ecosistema xUnit de EcoWaveProjectManagement.*
