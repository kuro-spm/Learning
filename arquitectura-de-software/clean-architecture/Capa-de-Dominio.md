# Capa de Dominio

## ¿Qué es?

La capa de dominio es el **centro** de Clean Architecture: contiene las entidades y las reglas de negocio puras de tu aplicación. Es la parte que describe *qué es* tu negocio (un pedido, un cliente, un producto) y *qué reglas debe cumplir siempre*, sin importar la tecnología que lo rodee.

## ¿Por qué existe?

Las reglas de negocio son lo más valioso y lo más estable de una aplicación: "un pedido sin líneas no se puede confirmar", "el stock nunca puede ser negativo". Estas reglas no deberían cambiar porque cambies de base de datos o de framework web.

La capa de dominio existe para **aislar** esas reglas en un lugar limpio, sin dependencias de nada externo. Aquí no hay SQL, ni HTTP, ni ficheros: solo objetos y lógica de negocio. Eso la hace fácil de entender, de probar y de proteger frente a los cambios técnicos.

> Si vienes de las bases de datos, no confundas una entidad de dominio con una tabla. La tabla es *cómo se guarda*; la entidad de dominio es *qué significa y qué reglas cumple*.

## ¿Cuándo y para qué se usa?

Es la primera capa que diseñas cuando modelas un negocio. En una tienda online, aquí viven `Pedido`, `Cliente`, `Producto` y las reglas que los gobiernan. En una app de reservas, vivirían `Reserva`, `Sala` y "no se puede reservar una sala ya ocupada".

Todo lo demás (casos de uso, base de datos, pantallas) se construye **alrededor** de esta capa y depende de ella, nunca al revés.

## Lo mínimo que necesitas saber

**1. Las entidades protegen sus propias reglas**

Una entidad no es una bolsa de datos pública: vigila que su estado sea siempre válido.

```csharp
public class Pedido
{
    private readonly List<LineaPedido> _lineas = new();
    public bool Confirmado { get; private set; }

    public void Confirmar()
    {
        if (_lineas.Count == 0)
            throw new InvalidOperationException("Un pedido sin líneas no se puede confirmar.");
        Confirmado = true;
    }
}
```

**2. La regla vive en la entidad, no fuera**

La validación "no se puede confirmar un pedido vacío" está *dentro* de `Pedido`. Así nadie puede saltarse la regla desde fuera.

**3. El dominio no depende de nada externo**

Nada de `using` hacia bases de datos, frameworks web o librerías de infraestructura. Si una entidad necesita un dato del exterior, se lo pasan ya calculado.

```csharp
// ❌ Una entidad de dominio NUNCA debería tener esto:
// using Microsoft.EntityFrameworkCore;
// using System.Net.Http;
```

**4. Objetos de valor para conceptos sin identidad**

Conceptos como un importe de dinero o un email se modelan como *objetos de valor*: pequeñas clases inmutables que también validan sus reglas.

```csharp
public record Email
{
    public string Valor { get; }
    public Email(string valor)
    {
        if (!valor.Contains('@'))
            throw new ArgumentException("Email no válido.");
        Valor = valor;
    }
}
```

## Lo que NO hace

- **No guarda ni lee datos** — no sabe que existe una base de datos; de eso se encargan los repositorios en la infraestructura.
- **No conoce la web ni la interfaz** — no sabe si lo llaman desde una API REST, una app de escritorio o un test.
- **No orquesta acciones completas** — coordinar "validar, cobrar y guardar" es trabajo de los casos de uso, no de las entidades.

## Buenas prácticas avanzadas

- **Haz imposible el estado inválido desde el constructor** — validar en los métodos no basta si se puede hacer `new Pedido()` vacío y rellenarlo a medias: la invariante ya nació rota. Usa constructores o factorías estáticas que exijan todo lo imprescindible (`Pedido.Crear(clienteId, lineas)`) y deja, como mucho, un constructor privado sin lógica para el ORM.
- **Distingue "regla de negocio incumplida" de "bug"** — que un pedido vacío no se pueda confirmar es un resultado *esperable* del negocio; devolverlo como un resultado explícito (patrón `Result`) permite a la capa de arriba reaccionar con normalidad. Reserva las excepciones para invariantes que nunca deberían romperse (si saltan, hay un bug). Usar excepciones para todo convierte el flujo normal de la aplicación en `try/catch` en cascada.
- **Ids tipados contra la obsesión por los primitivos** — con `int pedidoId` e `int clienteId`, el compilador acepta feliz que los intercambies al llamar a un método. Un objeto de valor mínimo lo convierte en error de compilación:

```csharp
public record IdPedido(int Valor);
public record IdCliente(int Valor); // ya no se pueden confundir
```

- **El reloj también es una dependencia** — un `DateTime.Now` dentro de una entidad hace la regla no determinista e imposible de probar ("el descuento caduca a medianoche"... ¿cómo lo testeas a las 10:00?). Pasa la fecha como parámetro (`pedido.Confirmar(fechaActual)`) o abstrae el reloj: el dominio no debería leer el mundo exterior ni para saber la hora.

---

*En resumen: la capa de dominio es el corazón limpio de tu aplicación — entidades y reglas de negocio que se protegen a sí mismas y no dependen de ninguna tecnología.*
