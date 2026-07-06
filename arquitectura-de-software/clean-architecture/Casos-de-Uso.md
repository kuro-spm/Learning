# Casos de Uso

## ¿Qué es?

Un caso de uso es una clase que representa **una acción concreta que tu aplicación sabe hacer**: "registrar un cliente", "confirmar un pedido", "cancelar una reserva". Vive en la capa de aplicación, justo alrededor del dominio, y se encarga de **orquestar** los pasos de esa acción.

## ¿Por qué existe?

Las entidades del dominio conocen sus propias reglas, pero una acción real suele necesitar **coordinar varias cosas**: leer datos, llamar a una entidad, guardar el resultado, enviar un aviso. Si ese baile se escribe dentro de un controlador web, la lógica de la aplicación queda atrapada en el framework y es imposible de reutilizar o probar.

Los casos de uso existen para tener esa coordinación en un sitio independiente del exterior. Cada caso de uso responde a la pregunta "¿qué pasos hay que dar para realizar esta acción?", apoyándose en el dominio para las reglas y en interfaces para todo lo demás.

> Si ya conoces el patrón "servicio de aplicación", un caso de uso es justo eso, pero acotado a **una sola acción** en lugar de agrupar muchas en un único servicio gigante.

## ¿Cuándo y para qué se usa?

Hay un caso de uso por cada cosa que un usuario (o el sistema) puede pedir a la aplicación. En una tienda online: `AñadirProductoAlCarrito`, `ConfirmarPedido`, `AplicarDescuento`. En un blog: `PublicarArtículo`, `ModerarComentario`.

Son el punto de entrada a la lógica: la capa web recibe la petición HTTP, la traduce y llama al caso de uso. El caso de uso hace el trabajo y devuelve un resultado simple, sin saber que vino de la web.

## Lo mínimo que necesitas saber

**1. Un caso de uso = una acción**

```csharp
public class ConfirmarPedido(IRepositorioPedidos repositorio)
{
    public async Task Ejecutar(int pedidoId)
    {
        var pedido = await repositorio.ObtenerPorId(pedidoId);
        pedido.Confirmar();                  // la regla vive en la entidad
        await repositorio.Guardar(pedido);
    }
}
```

**2. Orquesta, pero no contiene las reglas**

El caso de uso decide *el orden* de los pasos (leer → confirmar → guardar), pero la regla de "no confirmar un pedido vacío" sigue dentro de la entidad `Pedido`. El caso de uso coordina; el dominio decide.

**3. Depende de interfaces, no de tecnología**

Para guardar, cobrar o avisar, el caso de uso usa interfaces definidas en el centro. No sabe si por detrás hay SQL, un email real o un objeto falso de test.

```csharp
public class ProcesarCompra(IPasarelaPago pasarela, IRepositorioPedidos repositorio)
{
    public async Task Ejecutar(Pedido pedido, string tarjeta)
    {
        if (await pasarela.Cobrar(pedido.Total, tarjeta))
            await repositorio.Guardar(pedido);
    }
}
```

**4. Recibe y devuelve datos simples (DTOs)**

Para no acoplarse a la web, el caso de uso trabaja con objetos sencillos de entrada y salida, no con peticiones HTTP ni modelos de pantalla.

```csharp
public record DatosRegistro(string Nombre, string Email);
```

## Lo que NO hace

- **No contiene reglas de negocio profundas** — esas viven en el dominio; el caso de uso solo las invoca en el orden correcto.
- **No conoce la web ni la base de datos** — habla con el exterior solo a través de interfaces.
- **No gestiona detalles de transporte** — códigos de estado HTTP, rutas o serialización JSON son trabajo de la capa de adaptadores, no suyo.

## Buenas prácticas avanzadas

- **El caso de uso es el dueño de la transacción** — "leer, confirmar, guardar" debe ser todo o nada. La unidad de trabajo se abre y se cierra en el caso de uso, no dentro de cada repositorio: si cada `Guardar` hace commit por su cuenta, un fallo a mitad deja el pedido cobrado pero sin registrar. Es el error de consistencia más caro y más común.
- **Diseña para el doble clic: idempotencia** — todo caso de uso que llega por red acabará ejecutándose dos veces (reintentos, doble clic, timeouts con reenvío). `ConfirmarPedido` sobre un pedido ya confirmado no debería cobrar otra vez ni explotar: detecta la repetición y responde con un resultado claro. Quien no lo piensa se entera por un cobro duplicado en producción.
- **No encadenes casos de uso entre sí** — cuando `ConfirmarPedido` llama a `EnviarConfirmacion` (otro caso de uso), nace un grafo de orquestadores llamándose unos a otros que nadie puede seguir. Si dos casos de uso comparten pasos, extrae esos pasos a un servicio del dominio; si uno debe "enterarse" de lo que hizo otro, plantéate publicar un evento.
- **Centraliza lo transversal con decoradores** — logging, validación de la entrada, transacción y métricas repetidos a mano en cada caso de uso acaban desincronizados. Como todos los casos de uso tienen la misma forma (una clase, un método `Ejecutar`), puedes envolverlos con un decorador o un pipeline (los *behaviors* de MediatR son esto) y escribir cada preocupación transversal una sola vez.

---

*En resumen: un caso de uso orquesta los pasos de una acción concreta de la aplicación, apoyándose en el dominio para las reglas y en interfaces para el mundo exterior — una acción, una clase.*
