# Shouldly

## ¿Qué es?

Shouldly es una librería de aserciones para tests en .NET que reemplaza los métodos `Assert` estándar por una sintaxis fluida orientada al objeto bajo prueba, generando mensajes de error mucho más informativos cuando un test falla.

## ¿Por qué existe?

El problema con `Assert.AreEqual(expected, actual)` o `Assert.That(result, Is.EqualTo(expected))` (NUnit) es que cuando fallan, el mensaje de error dice qué salió mal pero no siempre deja claro *qué variable* tenía el valor incorrecto ni en qué contexto.

Shouldly invierte el foco: en lugar de `Assert.AreEqual(5, resultado)`, escribes `resultado.ShouldBe(5)`. Si falla, el mensaje incluye el nombre de la variable y su valor real, lo que acelera el diagnóstico sin tener que releer el test entero.

## ¿Cuándo y para qué se usa?

Shouldly se usa en los proyectos de tests de cualquier aplicación .NET: una tienda online, un sistema de facturación o una app de gestión de tareas. Cualquier test de servicio, repositorio o handler que verifique resultados de dominio utiliza Shouldly en lugar de las aserciones de xUnit o NUnit directamente. El objetivo es que cuando un test falle en CI, el mensaje sea autoexplicativo sin necesidad de abrir el código.

## Lo mínimo que necesitas saber

**1. Aserciones básicas de igualdad y nulidad**

```csharp
pedido.Numero.ShouldBe("PED-1001");
pedido.ShouldNotBeNull();
resultado.ShouldBeNull();
```

**2. Aserciones sobre colecciones**

```csharp
productos.ShouldNotBeEmpty();
productos.Count.ShouldBe(3);
productos.ShouldContain(p => p.Categoria == "Electrónica");
```

**3. Aserciones sobre excepciones**

```csharp
Should.Throw<ArgumentNullException>(() => servicio.CrearPedido(null));

var ex = Should.Throw<DomainException>(() => servicio.PublicarProducto(id));
ex.Message.ShouldContain("no autorizado");
```

**4. Aserciones sobre tipos**

```csharp
resultado.ShouldBeOfType<PedidoDto>();
handler.ShouldBeAssignableTo<IRequestHandler<CrearPedidoCommand>>();
```

**5. Instalación**

```xml
<PackageReference Include="Shouldly" Version="4.3.0" />
```

## Lo que NO hace

- **No es un framework de tests**: Shouldly solo aporta las aserciones; xUnit sigue siendo el runner que ejecuta los tests.
- **No reemplaza mocks**: para doblar dependencias se usa Moq o NSubstitute, no Shouldly.
- **No valida en tiempo de ejecución**: es exclusivamente para el entorno de tests, nunca en código de producción.

---

*En resumen: Shouldly hace que los tests fallidos hablen por sí solos, reduciendo el tiempo que pasas depurando por qué falló una aserción.*
