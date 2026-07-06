# Domain Events

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Un evento de dominio es un objeto inmutable que describe **algo que ya ha ocurrido** en el negocio y que puede interesar a otras partes del sistema: `PedidoConfirmado`, `StockAgotado`, `ClienteRegistrado`. Se nombra en pasado, porque describe un hecho consumado, no una orden.

## ¿Por qué existe?

Cuando confirmar un pedido implica también enviar un email de confirmación, avisar al almacén y actualizar estadísticas, la tentación es meter todas esas llamadas dentro del mismo caso de uso `ConfirmarPedido`. El problema es que ese caso de uso crece sin parar cada vez que aparece un nuevo interesado en el hecho, y encima acopla "confirmar un pedido" con "cómo se envía un email", dos responsabilidades que no tienen por qué conocerse.

Los eventos de dominio invierten la relación: el agregado (o el caso de uso) se limita a anunciar "esto ha pasado", sin saber ni preocuparse de quién reacciona. Los interesados se suscriben por su cuenta. En [Casos de Uso](../clean-architecture/Casos-de-Uso.md) ya se apuntaba esta idea al hablar de no encadenar casos de uso entre sí; esta ficha desarrolla el mecanismo concreto.

## ¿Cuándo y para qué se usa?

Cada vez que un hecho de negocio interesa a más de una parte del sistema: un pedido confirmado dispara el envío de un email, la actualización del stock y un registro de auditoría; un usuario registrado dispara un email de bienvenida y la creación de una configuración por defecto. Cuando ese hecho debe cruzar a otro servicio o proceso (no solo dentro de la misma aplicación), el evento de dominio se traduce en un **evento de integración**, que ya es otra pieza (colas, buses de mensajes).

## Lo mínimo que necesitas saber

**1. El evento describe un hecho pasado, y es inmutable**

```csharp
public record PedidoConfirmado(int PedidoId, DateTime FechaConfirmacion);
```

**2. El agregado registra el evento cuando ocurre el hecho**

```csharp
public class Pedido
{
    private readonly List<object> _eventos = new();
    public IReadOnlyCollection<object> EventosDominio => _eventos.AsReadOnly();

    public void Confirmar()
    {
        if (Lineas.Count == 0)
            throw new InvalidOperationException("Un pedido sin líneas no se puede confirmar.");

        Confirmado = true;
        _eventos.Add(new PedidoConfirmado(Id, DateTime.UtcNow));
    }

    public bool Confirmado { get; private set; }
    public IReadOnlyCollection<LineaPedido> Lineas { get; } = new List<LineaPedido>();
    public int Id { get; }
}
```

**3. Los eventos se publican después de guardar, no antes**

```csharp
public class ConfirmarPedido(IRepositorioPedidos repositorio, IPublicadorEventos publicador)
{
    public async Task Ejecutar(int pedidoId)
    {
        var pedido = await repositorio.ObtenerPorId(pedidoId);
        pedido.Confirmar();
        await repositorio.Guardar(pedido);          // 1. se persiste el hecho

        foreach (var evento in pedido.EventosDominio)
            await publicador.Publicar(evento);        // 2. se avisa después
    }
}
```

**4. Cada manejador (handler) hace una sola cosa, de forma independiente**

```csharp
public class EnviarEmailConfirmacion : IManejadorEvento<PedidoConfirmado>
{
    public Task Manejar(PedidoConfirmado evento) { /* enviar email */ return Task.CompletedTask; }
}

public class ActualizarEstadisticas : IManejadorEvento<PedidoConfirmado>
{
    public Task Manejar(PedidoConfirmado evento) { /* actualizar contador */ return Task.CompletedTask; }
}
```

Ninguno de los dos manejadores sabe que existe el otro, ni el caso de uso original sabe cuántos manejadores hay.

## Lo que NO hace

- **No es lo mismo que un evento de integración entre servicios** — el evento de dominio vive dentro del mismo proceso; cuando el hecho debe cruzar a otro servicio (vía un bus de mensajes o una cola), se traduce a un evento de integración aparte, normalmente con su propio contrato versionado.
- **No garantiza la entrega si se queda solo en memoria** — si el proceso se cae justo después de guardar pero antes de publicar, el evento se pierde; para garantizar que se publica sí o sí, se usa el patrón *Outbox* (guardar el evento en la misma transacción y publicarlo después desde ahí).
- **No debería contener lógica** — es un dato que describe un hecho, no un objeto con comportamiento.

## Buenas prácticas avanzadas

- **Publica siempre después de confirmar la transacción** — si publicas el evento y un manejador externo actúa (por ejemplo, cobra o envía un email) pero justo después el guardado falla y hace *rollback*, ese manejador ya reaccionó a un hecho que nunca llegó a ocurrir de verdad. El patrón *Outbox* resuelve esto guardando el evento como parte de la misma transacción de negocio y publicándolo en un segundo paso, con reintentos si hace falta.
- **Trata los manejadores como best-effort, no como parte del camino feliz** — si "enviar el email de confirmación" falla, normalmente no debería deshacer la confirmación del pedido. Aísla los fallos de cada manejador (captúralos, reintenta, encola) para que un problema en un interesado secundario no tumbe el flujo principal.
- **Vigila las cascadas de eventos síncronas** — un manejador que, al reaccionar, dispara otro evento, que dispara otro... reproduce el mismo problema que encadenar casos de uso (ver [Casos de Uso](../clean-architecture/Casos-de-Uso.md)), pero es más difícil de detectar porque nadie llama a nadie explícitamente en el código; solo se ve siguiendo el rastro de eventos en ejecución.
- **Versiona los eventos que salen del proceso** — un evento que solo vive en memoria se puede cambiar libremente, pero en cuanto se serializa (para un bus, un log o un *outbox*), añadir un campo nuevo es seguro y renombrar o quitar uno rompe a quien ya lo está leyendo. Trátalo como un contrato público desde el día en que cruza esa frontera.

---

*En resumen: un evento de dominio anuncia un hecho ya ocurrido para que otras partes del sistema reaccionen por su cuenta, sin que quien lo genera necesite saber quién escucha ni cuántos son.*
