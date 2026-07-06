# RPC

## ¿Qué es?

**RPC** (*Remote Procedure Call*, llamada a procedimiento remoto) es un estilo de API en el que el cliente invoca **funciones** que se ejecutan en otra máquina, como si fueran funciones locales. En lugar de pensar en recursos y URLs (como REST), piensas en **acciones**: `crearPedido()`, `enviarEmail()`, `calcularPrecio()`.

## ¿Por qué existe?

A veces tu API no encaja bien en "cosas con URL". Lo que tienes son **operaciones**: realizar un pago, reiniciar un servidor, traducir un texto. Forzar eso al molde de recursos REST resulta artificial. RPC nace de una idea muy natural para un programador: *llamar a una función que vive en otra máquina como si estuviera aquí*.

> Si ya sabes llamar a un método de tu propio código (`order.Cancel()`), RPC es eso mismo pero el método se ejecuta en un servidor remoto: tú llamas, el otro lado calcula y te devuelve el resultado.

Es uno de los estilos más antiguos de comunicación entre sistemas, y la base de la que beben [gRPC](gRPC.md) y, en parte, [SOAP](SOAP.md).

## ¿Cuándo y para qué se usa?

Encaja cuando la API es una colección de **acciones** más que de datos: una pasarela que ejecuta operaciones (`refund`, `capture`), microservicios internos que se llaman entre sí, o sistemas donde prima la simplicidad de "llamo a esta función con estos argumentos". Variantes muy conocidas son **JSON-RPC** y **XML-RPC** (según el formato que usen).

## Lo mínimo que necesitas saber

**1. Piensas en funciones, no en recursos**

La URL ya no nombra una "cosa", sino la acción a ejecutar.

```http
POST /api/sendEmail
POST /api/calculateShipping
POST /api/cancelOrder
```

**2. La petición lleva el nombre del método y sus parámetros**

Formato típico de **JSON-RPC**:

```json
POST /rpc
{
  "jsonrpc": "2.0",
  "method": "createOrder",
  "params": { "productId": 42, "quantity": 2 },
  "id": 1
}
```

**3. La respuesta trae el resultado de la función**

```json
{
  "jsonrpc": "2.0",
  "result": { "orderId": 87, "total": 59.80 },
  "id": 1
}
```

**4. Desde el cliente parece una llamada local**

Muchas librerías RPC generan código para que invocar el método remoto se sienta como invocar uno normal.

```csharp
// Por dentro viaja por la red, pero tú lo llamas como una función cualquiera
var result = await orderService.CreateOrderAsync(productId: 42, quantity: 2);
```

**5. Suele ir todo por un mismo verbo (normalmente POST)**

A diferencia de REST, RPC no aprovecha `GET`/`PUT`/`DELETE` para dar significado: la operación está en el *nombre del método*, no en el verbo HTTP.

## Lo que NO hace

- **No organiza por recursos** — si tu dominio son datos tipo CRUD, REST suele leerse mejor.
- **No usa los verbos ni los códigos HTTP de forma rica** — el significado va en el cuerpo, no en el protocolo.
- **No es un único estándar** — "RPC" es un estilo; JSON-RPC, XML-RPC, gRPC y SOAP son encarnaciones distintas con reglas propias.
- **No oculta del todo que la llamada es remota** — aunque lo parezca, sigue habiendo red de por medio: puede fallar, tardar o cortarse.

## Buenas prácticas avanzadas

- **No te creas la ilusión de "llamada local"** — es la trampa fundacional de RPC: como `orderService.CreateOrderAsync(...)` *parece* una función normal, se olvida que puede tardar 30 segundos, fallar a medias o no responder nunca. Toda llamada RPC necesita un *timeout* explícito y una decisión consciente sobre qué hacer al fallar. El código que trata lo remoto como local funciona perfecto... hasta el primer día de red inestable.
- **Diseña cada método pensando "¿qué pasa si se ejecuta dos veces?"** — cuando un RPC agota el tiempo de espera, no sabes si la función llegó a ejecutarse. Si el cliente reintenta `createOrder`, puedes acabar con dos pedidos. Los métodos de escritura deben aceptar un identificador de operación (`requestId`) que el servidor recuerde para responder lo mismo sin repetir el efecto, o diseñarse para que repetirlos sea inocuo.
- **Separa los errores de transporte de los errores de negocio** — "no pude contactar con el servidor" y "el cupón está caducado" son cosas distintas: la primera quizá se reintenta, la segunda jamás. JSON-RPC lo formaliza con rangos de códigos (los `-32xxx` reservados para el protocolo, el resto para tu aplicación); respétalo en vez de devolver todo como un error genérico, o el cliente no podrá decidir cuándo reintentar.
- **Huye del método comodín `execute(action, params)`** — cuando añadir métodos da pereza, aparece el RPC "genérico" que recibe el nombre de la operación como dato. Con eso pierdes todo lo bueno del estilo: el tipado, la documentación implícita del contrato y la posibilidad de evolucionar cada operación por separado. Si tu API tiene tres métodos y uno se llama `doAction`, tiene en realidad un número desconocido de métodos sin contrato.

---

*En resumen: RPC es llamar a una función que vive en otra máquina como si fuera local —piensas en acciones, no en recursos— y es la raíz de estilos modernos como gRPC.*
