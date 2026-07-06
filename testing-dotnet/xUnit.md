# xUnit

## ¿Qué es?

xUnit es el framework de testing más usado en .NET: define cómo se escriben los tests (`[Fact]`, `[Theory]`), cómo se ejecutan (`dotnet test`) y qué aserciones tienes disponibles (`Assert.Equal`, `Assert.Throws`...).

## ¿Por qué existe?

Sin un framework, "probar" es ejecutar la aplicación a mano y mirar si algo falla. xUnit convierte cada comprobación en un método que se ejecuta automáticamente, en aislamiento y con un veredicto binario: verde o rojo. Es el sucesor espiritual de NUnit y MSTest, escrito por los mismos autores de NUnit pero con decisiones más estrictas (por ejemplo: crea una instancia nueva de la clase de test para cada test, forzando el aislamiento).

> Si vienes del frontend, xUnit es a .NET lo que Vitest o Jest son a TypeScript: el runner, las aserciones y las convenciones, todo en uno.

## ¿Cuándo y para qué se usa?

Es la base de cualquier proyecto de tests en .NET moderno: los tests unitarios, los de integración y los de API se escriben todos con xUnit, aunque cambien las herramientas de apoyo (Testcontainers, `WebApplicationFactory`...). Se crea un proyecto separado (`MyApi.Tests`) que referencia al proyecto bajo test, y se ejecuta con `dotnet test` en local y en CI.

## Lo mínimo que necesitas saber

**1. `[Fact]`: un test sin parámetros**

Un método público decorado con `[Fact]` es un test. Si no lanza excepción, pasa.

```csharp
public class OrderTests
{
    [Fact]
    public void CalcularTotal_SinDescuento_SumaLosItems()
    {
        var order = new Order([new OrderItem(price: 10m, quantity: 3)]);

        Assert.Equal(30m, order.CalcularTotal());
    }
}
```

**2. `[Theory]`: el mismo test con varios datos**

Cuando el mismo escenario debe probarse con distintas entradas, `[Theory]` + `[InlineData]` evita duplicar el método:

```csharp
[Theory]
[InlineData("", false)]
[InlineData("sin-arroba", false)]
[InlineData("ana@example.com", true)]
public void EsEmailValido_SegunFormato_DevuelveLoEsperado(string email, bool expected)
{
    Assert.Equal(expected, EmailValidator.IsValid(email));
}
```

Para datos complejos (objetos, listas), existe `[MemberData]`, que toma los casos de una propiedad estática.

**3. Las aserciones básicas**

```csharp
Assert.Equal(expected, actual);      // igualdad (usa comparación estructural)
Assert.True(condition);              // condición booleana
Assert.NotNull(result);              // no es null
Assert.Contains("ADMIN", json);      // un string/colección contiene algo
Assert.Empty(errors);                // colección vacía
```

**4. Probar que algo lanza excepción**

`Assert.Throws<T>` ejecuta el código y verifica que lanza exactamente esa excepción — y te la devuelve para que asertes sobre sus detalles:

```csharp
var ex = Assert.Throws<InvalidOperationException>(() => order.Confirmar());
Assert.Contains("sin items", ex.Message);

// versión async:
await Assert.ThrowsAsync<UserNotFoundException>(() => service.GetUserAsync(unknownId));
```

**5. Tests asíncronos**

Un test puede ser `async Task` directamente; xUnit lo espera:

```csharp
[Fact]
public async Task GetUser_ConIdExistente_DevuelveElUsuario()
{
    var user = await repository.GetByIdAsync(existingId);
    Assert.NotNull(user);
}
```

**6. Nombres que cuentan una historia**

Una convención muy extendida es `Metodo_Escenario_ResultadoEsperado`. El nombre debe permitir entender qué se rompió sin abrir el archivo:

```csharp
POST_Login_ConContrasenaIncorrecta_Devuelve401()
Run_EnBaseDeDatosLimpia_AplicaMigracionesYEsIdempotente()
```

**7. Ejecutar los tests**

```bash
dotnet test                                    # toda la suite
dotnet test --filter "FullyQualifiedName~Login"  # solo los que contengan "Login"
```

## Lo que NO hace

- **No mockea dependencias** — para eso necesitas una librería aparte (NSubstitute, Moq).
- **No arranca tu API ni tu base de datos** — eso lo aportan `WebApplicationFactory` y Testcontainers; xUnit solo orquesta la ejecución.
- **No mide cobertura** — la cobertura la calcula un colector aparte como coverlet.
- **No garantiza orden de ejecución** — los tests se ejecutan en orden indefinido (y las clases, en paralelo por defecto); si un test depende de que otro se ejecute antes, está mal diseñado.

## Buenas prácticas avanzadas

- **xUnit crea una instancia nueva de la clase por cada test** — el constructor se ejecuta antes de *cada* test y `Dispose()` después de cada uno. Esto es un setup/teardown implícito: no necesitas atributos tipo `[SetUp]`. Quien no lo sabe pone estado compartido en campos de instancia esperando que persista entre tests, y no persiste (para compartir de verdad, usa fixtures — ver la ficha de fixtures).
- **Cuidado con `Assert.Equal` en decimales y fechas** — para `double`/`float` usa la sobrecarga con precisión (`Assert.Equal(0.3, result, precision: 5)`); para fechas, compara componentes o tolera un margen. Los tests que fallan "a veces" por redondeo o por milisegundos son de los más caros de diagnosticar.
- **Un `[Fact]` con bucle es una `[Theory]` mal escrita** — si un test itera sobre casos y aserta dentro del bucle, el primer fallo oculta los demás. Con `[Theory]` cada caso se ejecuta y reporta por separado.
- **Aprovecha el paralelismo, no luches contra él** — xUnit ejecuta clases de test en paralelo. Si dos clases comparten un recurso (una base de datos, un puerto), no desactives el paralelismo global: agrúpalas en una colección (`[Collection]`) para serializar solo esas.
- **`global using Xunit;` en el `.csproj`** — un `<Using Include="Xunit" />` en el proyecto de tests elimina el `using Xunit;` repetido en cada archivo. Detalle pequeño que delata un proyecto bien cuidado.

---

*En resumen: xUnit es el esqueleto de todo el testing en .NET — `[Fact]` para un caso, `[Theory]` para muchos, `Assert` para el veredicto, y `dotnet test` para ejecutarlo todo.*
