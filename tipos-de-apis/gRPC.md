# gRPC

## ¿Qué es?

**gRPC** es un framework moderno de [RPC](RPC.md) creado por Google que permite que los servicios se llamen entre sí invocando funciones remotas, usando un formato de datos binario y muy compacto llamado **Protocol Buffers** (*protobuf*). Está pensado para comunicación rápida y eficiente, sobre todo entre servicios de backend.

## ¿Por qué existe?

REST con JSON es legible y cómodo, pero el texto JSON ocupa espacio y hay que interpretarlo en cada extremo. Cuando tienes muchos microservicios hablándose miles de veces por segundo, ese coste se nota. gRPC ataca el problema con tres ideas: un formato **binario** (más pequeño y rápido que el texto), un **contrato fuerte** generado a partir de un fichero de definición, y el uso de **HTTP/2** (que permite varias llamadas a la vez por una sola conexión).

> Si JSON es como enviar una carta escrita a mano (clara pero voluminosa), protobuf es como enviar un formulario codificado: ocupa una fracción y la máquina del otro lado lo lee al instante.

## ¿Cuándo y para qué se usa?

Su terreno natural es la **comunicación interna entre microservicios**, donde importan la velocidad y el bajo consumo de red más que la legibilidad humana. Por ejemplo, un sistema de una tienda online donde el servicio de pedidos llama al de inventario y al de pagos cientos de veces por segundo. También se usa para comunicación en streaming. No es habitual exponerlo directamente a navegadores (necesitan una capa intermedia).

## Lo mínimo que necesitas saber

**1. El contrato se define en un fichero `.proto`**

Describes los servicios (funciones) y los mensajes (datos). De aquí se genera código para cliente y servidor en muchos lenguajes.

```proto
service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (Order);
}

message CreateOrderRequest {
  int32 product_id = 1;
  int32 quantity = 2;
}
```

**2. El código cliente se genera solo**

A partir del `.proto`, una herramienta crea las clases. Tú solo llamas al método.

```csharp
var client = new OrderService.OrderServiceClient(channel);
var order = await client.CreateOrderAsync(
    new CreateOrderRequest { ProductId = 42, Quantity = 2 });
```

**3. Los datos viajan en binario (protobuf)**

No verás JSON legible en la red: el mensaje va codificado y compacto, lo que lo hace rápido de enviar y de leer.

**4. Soporta streaming en ambos sentidos**

Además de la clásica petición-respuesta, una llamada gRPC puede mantener un flujo continuo de mensajes (del cliente, del servidor o de ambos a la vez).

```proto
rpc WatchPrices (PriceRequest) returns (stream PriceUpdate);
```

**5. Se apoya en HTTP/2**

Esto le permite multiplexar (varias llamadas por una conexión) y reducir latencia frente al HTTP/1.1 típico de REST.

## Lo que NO hace

- **No es legible a ojo** — al ser binario, no puedes inspeccionarlo fácilmente como un JSON en el navegador.
- **No funciona directo en navegadores** — el navegador no controla HTTP/2 a ese nivel; hace falta una capa intermedia (gRPC-Web).
- **No es la mejor opción para APIs públicas** — exige que el cliente comparta el `.proto` y herramientas concretas; para terceros, REST o GraphQL son más accesibles.
- **No elimina la complejidad** — el tooling y la curva de aprendizaje son mayores que los de una API REST sencilla.

---

*En resumen: gRPC es RPC con esteroides —contrato fuerte, datos binarios y HTTP/2— pensado para que los microservicios se hablen entre sí a máxima velocidad.*
