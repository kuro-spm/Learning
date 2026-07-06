# NSubstitute

**¿Qué es?** Una librería de *mocking* para .NET: crea implementaciones falsas de interfaces sobre la marcha, para que un test unitario pueda aislar la clase bajo prueba de sus dependencias reales (base de datos, email, APIs externas). Es la alternativa moderna a Moq, con una sintaxis más limpia.

---

## ¿Por qué existe?

Para probar `OrderService` sin NSubstitute tendrías que escribir a mano una clase `FakeProductRepository` con contadores y flags para saber qué se llamó — una por cada test distinto. NSubstitute genera esos dobles en una línea y te deja configurar respuestas y verificar llamadas con métodos de extensión que se leen casi como prosa.

> Si conoces Vitest o Jest, `Substitute.For<T>()` es su `vi.fn()` / `jest.mock()`, pero tipado: el sustituto cumple la interfaz completa y el compilador vigila.

---

## ¿Cuándo y para qué se usa?

En **tests unitarios**, para las dependencias que no son el objetivo del test: el servicio de pedidos se prueba de verdad, pero el repositorio y el notificador son sustitutos. El patrón encaja especialmente bien con arquitecturas de puertos y adaptadores: los *puertos* (interfaces) se sustituyen, y la lógica de negocio se ejercita pura y rápida.

No se usa en tests de integración — ahí el punto es precisamente usar las piezas reales.

---

## Lo mínimo que necesitas saber

**1. Crear un sustituto y configurar respuestas**

```csharp
var repository = Substitute.For<IProductRepository>();
repository.GetByIdAsync(42).Returns(new Product(42, "Teclado", 89.90m));

var service = new OrderService(repository);
```

**2. Verificar que se llamó (o que NO se llamó)**

```csharp
await service.CancelOrderAsync(orderId);

await repository.Received(1).DeleteAsync(orderId);
await emailSender.DidNotReceive().SendAsync(Arg.Any<Email>());
```

**3. Argumentos flexibles con `Arg`**

```csharp
repository.SearchAsync(Arg.Is<string>(q => q.Contains("teclado")))
          .Returns([new Product(42, "Teclado", 89.90m)]);
```

**4. Simular errores de la dependencia**

```csharp
repository.GetByIdAsync(Arg.Any<int>())
          .ThrowsAsync(new TimeoutException());

await Assert.ThrowsAsync<TimeoutException>(() => service.GetOrderAsync(1));
```

---

## Lo que NO hace

- **No sustituye clases con métodos no virtuales** — funciona sobre interfaces (y miembros virtuales). Si tu diseño no tiene interfaces en las costuras, primero hay que refactorizar.
- **No prueba la implementación real de la dependencia** — el SQL del repositorio real sigue sin probar; eso es trabajo de los tests de integración.
- **No detecta que el mock miente** — si configuras el sustituto con un comportamiento que la implementación real no tiene, el test pasa y producción falla.

---

## Buenas prácticas avanzadas

- **Verifica interacciones solo cuando la interacción ES el contrato** — `Received()` sobre cada llamada interna acopla el test a la implementación y lo rompe con cada refactor. Aserta sobre el *resultado* (lo que devuelve, lo que persiste) siempre que puedas; reserva `Received()` para efectos que no se observan de otra forma (que se envió el email, que se publicó el evento).
- **Si necesitas mockear más de 3-4 dependencias, el problema es la clase, no el test** — un Arrange plagado de sustitutos es el síntoma clásico de una clase con demasiadas responsabilidades.
- **Cuidado con los sustitutos que devuelven el default silenciosamente** — un método no configurado devuelve `null`/default sin avisar, y el test puede pasar por el camino equivocado. Configura explícitamente todo lo que el flujo bajo prueba vaya a tocar.

---

*En resumen: NSubstitute fabrica las dependencias falsas para que el test unitario pruebe solo tu lógica — configura lo que devuelven, verifica lo imprescindible y no mockees lo que deberías probar de verdad.*
