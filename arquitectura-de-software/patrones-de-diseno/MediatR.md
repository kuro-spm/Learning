# MediatR

## ¿Qué es?

MediatR es una librería de .NET que implementa el patrón [Mediator](Mediator.md): en lugar de que una clase llame directamente a otra, se define un *mensaje* (un objeto con los datos de lo que se quiere hacer) y se envía a un mediador central, que lo entrega al *handler* que sabe procesarlo. Quien envía el mensaje nunca conoce a quien lo ejecuta.

## ¿Por qué existe?

Lo natural, cuando una pieza del sistema necesita disparar la lógica de otra, es que la primera tenga una referencia a la segunda y la llame. Funciona, pero acopla las dos: la que llama tiene que conocer la clase concreta de la que ejecuta, sus dependencias y su constructor. En una aplicación organizada en módulos esto se agrava, porque para que un módulo llame a otro tendría que referenciarlo como proyecto, y en cuanto dos módulos se necesitan mutuamente aparece una **dependencia circular** que no compila.

MediatR rompe ese acoplamiento poniendo un intermediario en medio. El emisor solo conoce el mensaje y al mediador; el mediador ya sabe (por convención, escaneando los tipos registrados) qué handler corresponde a cada mensaje.

> Piensa en la recepcionista de una oficina grande: tú no vas a buscar despacho por despacho a la persona que resuelve tu gestión; entregas tu solicitud en recepción y esta ya sabe a qué departamento enviarla. No necesitas conocer la estructura interna de la empresa, solo hablar con recepción.

Antes de meter MediatR, conviene preguntarse si de verdad hace falta. Si solo hay una o dos interacciones sencillas, una **interfaz normal en una capa compartida**, implementada en otro sitio e inyectada por DI, resuelve el desacoplamiento sin añadir ninguna librería nueva. MediatR gana cuando las comunicaciones se multiplican, cuando quieres un punto único para aplicar comportamiento común, o cuando son módulos enteros los que no deben referenciarse.

## ¿Cuándo y para qué se usa?

Tres escenarios típicos, todos dentro de un mismo proceso:

- **CQRS in-process** — separar las operaciones que cambian datos (*commands*, p. ej. `CreateOrderCommand`) de las que solo leen (*queries*, p. ej. `GetProductByIdQuery`). Cada una es un mensaje con su handler; el controlador solo hace `Send` y no sabe qué clase lo resuelve. En una tienda online, el endpoint de "crear pedido" envía un command; el de "ver ficha de producto" envía una query.
- **Comunicación entre módulos** — en un *modular monolith* (un blog con módulos de "contenido", "comentarios" y "notificaciones", por ejemplo), un módulo puede publicar un mensaje y otro reaccionar sin que ninguno referencie el proyecto del otro. Se evitan las dependencias circulares y cada módulo se puede razonar por separado.
- **Pipelines de comportamiento** — aplicar lógica transversal (logging, validación, medición de tiempos) a todos los mensajes de golpe, sin repetirla en cada handler. En una app de tareas, un único *behavior* de validación puede rechazar toda petición mal formada antes de que llegue a su handler.

## Lo mínimo que necesitas saber

**1. Definir el mensaje: `IRequest<T>`**

Un mensaje es un objeto normal (un `record` va perfecto) que declara qué devuelve al ser procesado.

```csharp
// Una query que devuelve un producto
public record GetProductByIdQuery(int ProductId) : IRequest<ProductDto>;

// Un command que devuelve el id del pedido creado
public record CreateOrderCommand(int CustomerId, List<int> ProductIds) : IRequest<int>;
```

**2. Definir el handler: `IRequestHandler<TRequest, TResponse>`**

Hay **exactamente un** handler por cada `IRequest`. Recibe sus dependencias por el constructor, como cualquier servicio.

```csharp
public class GetProductByIdHandler(IProductRepository repository)
    : IRequestHandler<GetProductByIdQuery, ProductDto>
{
    public async Task<ProductDto> Handle(GetProductByIdQuery request, CancellationToken cancellationToken)
    {
        var product = await repository.GetById(request.ProductId);
        return new ProductDto(product.Id, product.Name, product.Price);
    }
}
```

**3. Inyectar `IMediator` y enviar el mensaje con `Send`**

Quien envía nunca hace `new GetProductByIdHandler(...)`: solo conoce el mensaje y al mediador.

```csharp
public class ProductsController(IMediator mediator) : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var product = await mediator.Send(new GetProductByIdQuery(id));
        return Ok(product);
    }
}
```

**4. Registrar MediatR en el arranque**

Se le indica qué *assembly* debe escanear para descubrir automáticamente todos los handlers.

```csharp
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(GetProductByIdQuery).Assembly));
```

**5. Comportamiento común con *pipeline behaviors***

Un *behavior* envuelve a **todos** los mensajes; `next()` llama al siguiente eslabón (otro behavior o, al final, el handler).

```csharp
public class LoggingBehavior<TRequest, TResponse>(ILogger<TRequest> logger)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        logger.LogInformation("Procesando {Request}", typeof(TRequest).Name);
        return await next();
    }
}
```

## Lo que NO hace

- **No es un bus de mensajería distribuido** — todo ocurre en memoria, dentro del mismo proceso. Un mensaje de MediatR no cruza la red ni llega a otro servicio.
- **No sustituye a un *message broker*** — si necesitas comunicación entre servicios separados, colas duraderas o *publish/subscribe* real, eso es territorio de un bus como MassTransit sobre RabbitMQ o Azure Service Bus (ver [Mensajería Asíncrona](../../devops/mensajeria-asincrona/Mensajeria-Asincrona.md)), no de MediatR.
- **No da persistencia ni reintentos** — si el proceso muere a mitad, el mensaje se pierde; no hay reenvío automático ni *dead-letter queue*.
- **No mejora el rendimiento** — por debajo sigue siendo una llamada a método (con un poco de indirección extra). Lo que aporta es desacoplamiento, no velocidad.

## Buenas prácticas avanzadas

- **Fija la versión conscientemente: la licencia cambió** — MediatR fue MIT (gratis) hasta la **v12**. A partir de la **v13** pasó a una **licencia comercial** para uso empresarial. No es un detalle menor: revisa qué versión arrastra tu proyecto y, si dependes de la versión gratuita, fija el rango en el `.csproj` (p. ej. `Version="[12.*,13.0.0)"`) para no saltar a una versión de pago sin darte cuenta en un `dotnet restore`.
- **Concentra lo transversal en *behaviors*, no en cada handler** — logging, validación (con FluentValidation), transacciones o métricas viven mucho mejor en el pipeline que copiados en cada `Handle`. Y **el orden importa**: si el behavior de validación se registra después del de logging, los mensajes inválidos aparecerán en el log antes de ser rechazados. Ordena con intención, no según el orden en que los vas añadiendo.
- **No lo uses para llamadas triviales** — envolver en un mensaje + handler algo que una clase ya podría llamar directamente añade indirección (buscar el tipo, registrar el mensaje, saltar al mediador) sin beneficio real. MediatR merece la pena cuando desacopla un controlador de su lógica o un módulo de otro; dentro de una sola clase pequeña es *over-engineering*. Si dudas, empieza con una **interfaz simple inyectada por DI** y sube a MediatR solo cuando el número de interacciones lo justifique.
- **Mantén mensaje y handler cohesionados y de una sola responsabilidad** — como no hay una llamada directa que "avise" del tamaño, es fácil que un handler crezca hasta orquestar media aplicación. Un handler es un caso de uso: si necesita hacer demasiadas cosas, probablemente sean varios mensajes.
- **Vigila el escaneo de *assemblies*** — `RegisterServicesFromAssembly` reflexiona sobre los tipos al arrancar. En una solución grande con muchos ensamblados, registrar todo indiscriminadamente encarece el *startup*; sé explícito sobre qué *assemblies* escaneas en lugar de barrer el dominio entero.
- **Testea los handlers directamente** — un handler es una clase normal con sus dependencias inyectadas; instanciarlo y llamar a `Handle` en un test es más simple y rápido que montar todo el pipeline de MediatR solo para comprobar su lógica.

## Recursos didácticos

- Repositorio oficial y wiki: [github.com/jbogard/MediatR](https://github.com/jbogard/MediatR) — el `README` y la wiki cubren requests, notificaciones y behaviors con ejemplos mínimos.
- Para entender el patrón que hay debajo antes de la librería, lee primero [Mediator](Mediator.md) en esta misma colección.

---

*En resumen: MediatR es la implementación .NET del patrón Mediator — enruta mensajes a sus handlers dentro del mismo proceso para desacoplar quién pide algo de quién lo hace, ideal para CQRS y comunicación entre módulos, pero nunca un sustituto de un bus de mensajería distribuido (y ojo con su licencia a partir de la v13).*
