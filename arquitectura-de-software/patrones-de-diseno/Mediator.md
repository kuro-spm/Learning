# Mediator

## ¿Qué es?

Mediator es un patrón de diseño (GoF) en el que un objeto central —el *mediador*— se encarga de toda la comunicación entre otros objetos, para que estos no se referencien directamente entre sí. En .NET, la librería **MediatR** es la implementación más extendida: cada acción se modela como un mensaje que se envía al mediador, y este lo entrega a quien sabe manejarlo.

## ¿Por qué existe?

Cuando dos piezas del sistema necesitan hablarse, la solución directa es que una llame a la otra: un controlador que instancia un servicio, un servicio que llama a otro servicio. Funciona bien al principio, pero en cuanto crecen las piezas y las comunicaciones, cada una acaba conociendo a muchas otras, y cualquier cambio en una obliga a revisar quién la llama y cómo.

El Mediator invierte esa relación: nadie llama a nadie directamente, todos hablan con el mediador, y es el mediador quien decide (o ya sabe, por convención) quién procesa cada mensaje. Quien envía el mensaje no conoce ni necesita conocer a quien lo procesa.

> Piensa en una torre de control aérea: los aviones no se coordinan entre ellos por radio directa para no chocar; todos hablan con la torre, y es la torre quien decide el orden de aterrizaje. Ningún avión necesita conocer la posición de los demás, solo escuchar a la torre.

## ¿Cuándo y para qué se usa?

Cuando un controlador (o cualquier punto de entrada) necesita disparar una acción de negocio sin acoplarse a la clase concreta que la ejecuta: en vez de instanciar el caso de uso directamente, envía un mensaje al mediador y este lo entrega al manejador correspondiente. También aparece quan una misma acción interesa a varios manejadores independientes (notificaciones tipo publicar/suscribir dentro del mismo proceso), o para aplicar comportamiento común (logging, validación) a todos los mensajes sin tocar cada manejador uno a uno.

En una aplicación organizada en módulos, el Mediator es además una forma de que dos módulos se comuniquen sin referenciarse como proyectos: un módulo publica un mensaje y otro lo escucha, sin que ninguno de los dos conozca la clase concreta del otro lado (ver [Módulos vs Shared](../clean-architecture/Modulos-vs-Shared.md)).

## Lo mínimo que necesitas saber

**1. Petición con una única respuesta: `IRequest` / `IRequestHandler`**

```csharp
public record ConfirmarPedidoCommand(int PedidoId) : IRequest<bool>;

public class ConfirmarPedidoHandler(IRepositorioPedidos repositorio) : IRequestHandler<ConfirmarPedidoCommand, bool>
{
    public async Task<bool> Handle(ConfirmarPedidoCommand request, CancellationToken cancellationToken)
    {
        var pedido = await repositorio.ObtenerPorId(request.PedidoId);
        pedido.Confirmar();
        await repositorio.Guardar(pedido);
        return true;
    }
}
```

Hay exactamente **un** manejador por petición.

**2. Quien envía el mensaje no conoce al manejador**

```csharp
public class PedidosController(IMediator mediator) : ControllerBase
{
    [HttpPost("{id}/confirmar")]
    public async Task<IActionResult> Confirmar(int id)
    {
        var resultado = await mediator.Send(new ConfirmarPedidoCommand(id));
        return Ok(resultado);
    }
}
```

El controlador nunca hace `new ConfirmarPedidoHandler(...)`; solo conoce el mensaje y al mediador.

**3. Notificaciones: `INotification` / `INotificationHandler`, muchos manejadores por mensaje**

```csharp
public record PedidoConfirmadoNotification(int PedidoId) : INotification;

public class EnviarEmailConfirmacion : INotificationHandler<PedidoConfirmadoNotification> { /* ... */ }
public class ActualizarEstadisticas : INotificationHandler<PedidoConfirmadoNotification> { /* ... */ }

await mediator.Publish(new PedidoConfirmadoNotification(pedidoId));
// ambos manejadores se ejecutan, sin conocerse entre sí
```

**4. Comportamiento común a través de *pipeline behaviors***

```csharp
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        logger.LogInformation("Procesando {Request}", typeof(TRequest).Name);
        return await next();
    }
}
```

Un *behavior* envuelve a **todos** los mensajes registrados, sin tocar cada manejador uno a uno.

## Lo que NO hace

- **No es un bus de mensajes entre procesos o servicios** — todo ocurre en memoria, dentro del mismo proceso. Si el mensaje necesita salir a otro servicio o sobrevivir a un reinicio, hace falta una cola o un broker real (ver [Mensajería Asíncrona](../../devops/mensajeria-asincrona/Mensajeria-Asincrona.md)), no MediatR.
- **No sustituye a los eventos de dominio intra-módulo** — cuando el hecho solo interesa dentro del mismo módulo, puede publicarse de forma local sin pasar por un mediador compartido por toda la aplicación (ver [Domain Events](Domain-Events.md)).
- **No mejora el rendimiento** — por debajo sigue siendo una llamada a método; lo que gana es desacoplamiento, no velocidad.

## Buenas prácticas avanzadas

- **Resérvalo para fronteras que de verdad lo necesitan** — usarlo para toda comunicación dentro de un mismo módulo pequeño añade una capa de indirección (buscar el manejador, registrar el mensaje) sin beneficio real; el patrón brilla cuando desacopla un controlador de su lógica, o un módulo de otro, no dentro de una clase que ya podría llamar a otra directamente.
- **No dejes que un manejador se convierta en un caso de uso gigante** — es fácil que, al no haber una llamada directa que "avise" del tamaño, un manejador acabe orquestando demasiadas responsabilidades. Las mismas reglas de una buena orquestación (ver [Casos de Uso](../clean-architecture/Casos-de-Uso.md)) siguen aplicando dentro de un `Handle`.
- **El orden de los *pipeline behaviors* importa** — un behavior de validación registrado después de uno de logging deja pasar mensajes inválidos por el log antes de rechazarlos; decide el orden con intención, no por el orden en que los vas registrando.
- **Testea los manejadores de forma directa, no a través del mediador** — un manejador es una clase normal con sus dependencias inyectadas; instanciarlo y llamar a `Handle` en un test es más simple y más rápido que montar todo el pipeline de MediatR para comprobar su lógica.

---

*En resumen: Mediator hace que quien pide una acción y quien la ejecuta no se conozcan entre sí — todos hablan con un mediador central, lo que permite añadir manejadores, comportamiento común o desacoplar módulos sin tocar el código que envía el mensaje.*
