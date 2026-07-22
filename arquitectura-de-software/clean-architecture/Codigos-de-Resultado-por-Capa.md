# Códigos de Resultado por Capa

## ¿Qué es?

Un **código de resultado** es un valor —casi siempre un `enum`— que nombra **cómo terminó** una operación de la aplicación: `OK`, `NO_ENCONTRADO`, `STOCK_INSUFICIENTE`, `PAGO_RECHAZADO`… Esta ficha no trata el patrón en sí (eso es el [Result Pattern](../patrones-de-diseno/Result-Pattern.md)), sino una decisión que confunde a mucha gente: **en qué capa de Clean Architecture debe vivir ese enum, y por qué**.

## ¿Por qué existe?

En cuanto adoptas el Result Pattern y modelas el motivo del fallo como un enum (en vez de un texto suelto), aparece la pregunta: *¿dónde pongo `MotivoFallo`?* Hay dos respuestas intuitivas y **las dos suelen estar mal**:

- *"Donde nace el error"* → en Infraestructura, que es quien ve el `429 Too Many Requests` del proveedor de pago.
- *"Con las demás cosas del negocio"* → en el Dominio, junto a las entidades y sus enums.

Ambas rompen la [Regla de Dependencia](Regla-de-Dependencia.md). El código de resultado es el **vocabulario con el que un caso de uso comunica su desenlace** a quien lo llama; por tanto pertenece a la **capa de aplicación**, que es donde viven los [casos de uso](Casos-de-Uso.md). Ni más adentro (Dominio) ni más afuera (Infraestructura o web).

La clave es distinguir **dos tipos de enum**, porque no todos van al mismo sitio:

| Tipo de enum | Representa | Capa | Ejemplo |
|---|---|---|---|
| **De dominio** | *Qué es* algo — un concepto intrínseco del negocio | Dominio | `EstadoPedido`, `TipoCliente` |
| **De resultado** | *Cómo terminó* una operación | Aplicación | `ConfirmarPedidoResultado`, `MotivoFallo` |

`EstadoPedido.ENVIADO` describe el negocio y lo usan las entidades: es de dominio. `PAGO_RECHAZADO` describe cómo salió *ejecutar una acción* orquestando un cobro externo: es de resultado, y va en Aplicación.

## ¿Cuándo y para qué se usa?

En cualquier aplicación cuyos casos de uso devuelven un resultado tipado en lugar de lanzar excepciones para los fallos esperados. En una tienda online, `ConfirmarPedido` puede acabar en `OK`, `PEDIDO_VACIO`, `STOCK_INSUFICIENTE` o `PAGO_RECHAZADO`; en un blog, `PublicarArtículo` en `OK`, `BORRADOR_VACIO` o `SIN_PERMISOS`. Ese enum de desenlace es el que colocas en la capa de aplicación, y las capas de alrededor lo **traducen** en cada dirección: la de fuera (web) lo convierte en un código HTTP; la de dentro-técnica (adaptadores) traduce sus errores crudos *hacia* él.

## Lo mínimo que necesitas saber

**1. El enum de resultado y el caso de uso viven juntos, en Aplicación**

```csharp
// Capa de Aplicación
public enum ConfirmarPedidoResultado { Ok, PedidoVacio, StockInsuficiente, PagoRechazado }

public class ConfirmarPedido(IRepositorioPedidos repo, IPasarelaPago pasarela)
{
    public async Task<Result<Pedido, ConfirmarPedidoResultado>> Ejecutar(int pedidoId)
    {
        var pedido = await repo.ObtenerPorId(pedidoId);
        if (pedido.EstaVacio)
            return Result.Fail(ConfirmarPedidoResultado.PedidoVacio);
        // ...
    }
}
```

**2. La Infraestructura traduce sus errores crudos HACIA el código (hacia dentro)**

El adaptador de la pasarela es el único que ve el `429`/`402`/timeout del proveedor. Su trabajo es mapear ese detalle técnico al vocabulario de la aplicación, **sin dejar que la excepción cruda cruce la frontera**:

```csharp
// Capa de Infraestructura — implementa una interfaz definida en el centro
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests)
{
    return ConfirmarPedidoResultado.PagoRechazado; // el 429 no sale de aquí
}
```

Esto solo es posible si el enum está en Aplicación: Infraestructura ya depende de Aplicación (regla de dependencia), así que puede referenciarlo. Al revés sería imposible.

**3. La capa web traduce DESDE el código a HTTP (hacia fuera)**

El controlador convierte el desenlace en el protocolo concreto de esa frontera. El caso de uso no sabe que existe HTTP:

```csharp
// Capa de Presentación (web)
return resultado.Motivo switch
{
    ConfirmarPedidoResultado.Ok               => Ok(resultado.Value),
    ConfirmarPedidoResultado.PedidoVacio      => BadRequest(),
    ConfirmarPedidoResultado.StockInsuficiente => Conflict(),
    ConfirmarPedidoResultado.PagoRechazado    => StatusCode(402),
    _                                          => StatusCode(500)
};
```

**4. La dirección de las dependencias es lo que fija el sitio**

Infraestructura y web **apuntan hacia** Aplicación; Aplicación no apunta a ninguna de las dos. Como las dos necesitan conocer el enum (una para producirlo, otra para consumirlo), el único sitio que ambas pueden ver sin violar la regla es la capa a la que ambas apuntan: Aplicación.

```
Infraestructura ──► Aplicación (enum) ◄── Presentación (web)
     traduce hacia           ▲            traduce desde
```

## Lo que NO hace

- **No decide el código HTTP** — el enum es agnóstico del transporte; el mapeo a `400`/`404`/`409` es trabajo de la capa web (o de la cola, o del gRPC…), no del enum.
- **No es un enum de dominio** — no describe qué es una entidad, sino cómo terminó una acción; por eso no va en el Dominio junto a `EstadoPedido`.
- **No deja escapar excepciones crudas** — un `SqlException` o un `429` nunca deben cruzar hacia fuera como tales; el adaptador los convierte al código antes de devolver.

## Buenas prácticas avanzadas

- **Un enum de resultado por caso de uso (o por módulo), no uno global** — un `ErrorCode` gigante compartido por toda la app acaba con valores que solo aplican a una operación y obliga a cada llamador a ignorar la mitad. Acota el enum al desenlace real de esa acción: quien llama a `ConfirmarPedido` solo debería ver desenlaces de confirmar pedidos.
- **No metas el protocolo dentro del enum** — nombrar el valor `Error401` o `NotFound404` mete HTTP en la capa de aplicación y la ata a la web. Nombra por **motivo de negocio** (`SinPermisos`, `NoEncontrado`) y deja el número para la traducción del borde. Puedes documentar el mapeo esperado en un comentario, pero el enum no debe depender de HTTP.
- **La traducción va en los adaptadores, en las dos direcciones** — el de entrada (web) traduce *desde* el código; el de salida (cliente externo) traduce *hacia* él. Si ves un `switch` sobre códigos HTTP dentro de un caso de uso, la traducción se ha colado en la capa equivocada.
- **Errores como valores solo para lo esperado** — el código de resultado es para fallos que forman parte del flujo (`StockInsuficiente`); un bug o una dependencia caída siguen siendo una excepción. Si mezclas ambos para el mismo problema, quien llama tiene que protegerse por partida doble (ver [Result Pattern](../patrones-de-diseno/Result-Pattern.md)).
- **Si el enum lo comparten varios módulos, sube de sitio con cuidado** — un desenlace específico de un módulo se queda en la aplicación de ese módulo; solo si de verdad es transversal va a un proyecto de aplicación compartido, nunca al Dominio compartido (ver [Módulos vs Shared](Modulos-vs-Shared.md)).

## Recursos didácticos

- [http.cat](https://http.cat/) — cada código de estado HTTP ilustrado con un gato. Útil para fijar a qué se traduce cada desenlace en la capa web (`404`, `409`, `422`…) de forma que no se olvide.
- [Regla de Dependencia](Regla-de-Dependencia.md) y [Puertos y Adaptadores](Puertos-y-Adaptadores.md) — las dos ideas de esta colección sobre las que se apoya toda esta decisión.

---

*En resumen: el código de resultado es el vocabulario con el que un caso de uso dice cómo terminó, así que vive en la capa de aplicación — la Infraestructura traduce sus errores crudos hacia él y la web lo traduce desde él a HTTP, y esa doble traducción es justo lo que impone la regla de dependencia.*
