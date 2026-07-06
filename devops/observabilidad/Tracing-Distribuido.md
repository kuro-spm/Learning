# Tracing Distribuido

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

El *tracing* distribuido es la técnica que permite seguir el recorrido completo de una única petición a través de varios servicios, registrando cuánto ha tardado cada tramo del camino. El resultado se llama **traza** (*trace*), y cada tramo dentro de ella es un **span**.

## ¿Por qué existe?

En un sistema con un solo servicio, saber por qué una petición tarda 2 segundos es cuestión de mirar su propio log o hacer *profiling* local. En un sistema distribuido —una API que llama a un servicio de pagos, que a su vez consulta a un proveedor externo— esa misma pregunta implica reconstruir a mano qué pasó en tres sistemas distintos, con logs separados y sin nada que los conecte entre sí.

El tracing distribuido resuelve exactamente eso: asigna un identificador único a la petición (`traceId`) que viaja con ella de servicio en servicio, y cada servicio añade su propio tramo (`span`) con cuánto ha tardado y qué ha hecho. Al final puedes reconstruir el camino completo como si fuera uno solo.

> Piensa en el número de seguimiento de un paquete: cada almacén por el que pasa añade un sello con la hora, pero es el mismo número el que te permite ver el trayecto completo de principio a fin, aunque lo hayan tocado empresas distintas.

## ¿Cuándo y para qué se usa?

Imprescindible en cuanto una petición cruza más de un servicio: una tienda online donde confirmar un pedido implica al servicio de pedidos, al de pagos y al de inventario, y quieres saber cuál de los tres está ralentizando el checkout. También sirve para ver dependencias que ni siquiera conocías (un endpoint que, sin que nadie lo recuerde, acaba llamando a un servicio de terceros).

## Lo mínimo que necesitas saber

**1. Una traza es un árbol de spans**

```
Trace: "Confirmar pedido #87"          (350 ms total)
├─ Span: pedidos.CrearPedido           (120 ms)
│   └─ Span: pedidos.GuardarEnBD       (40 ms)
├─ Span: pagos.Cobrar                  (80 ms)
└─ Span: inventario.ReservarStock      (150 ms)
```

**2. Cada span tiene un identificador y un padre**

```csharp
using var span = tracer.StartActiveSpan("pagos.Cobrar");
span.SetAttribute("orderId", pedido.Id);
span.SetAttribute("metodoPago", "tarjeta");
```

**3. El contexto viaja entre servicios en las cabeceras**

Cuando el servicio de pedidos llama por HTTP al de pagos, el `traceId` y el `spanId` actual viajan en una cabecera estándar (`traceparent`), para que el servicio de pagos sepa a qué traza pertenece su propio span.

```
traceparent: 00-abc123def456...-789span...-01
```

**4. Los spans llevan atributos y eventos**

Además del tiempo, un span puede llevar metadatos (`orderId`, `metodoPago`) y eventos puntuales dentro de su duración (como una excepción capturada), que ayudan a entender qué pasó exactamente en ese tramo.

**5. Se visualiza como una línea de tiempo (*waterfall*)**

Las herramientas de tracing (Jaeger, Zipkin, Application Insights) muestran los spans como barras apiladas en el tiempo, de forma que se ve a simple vista cuál es el tramo que más tarda.

## Lo que NO hace

- **No sustituye a los logs** — una traza dice *cuánto* tardó cada tramo, no el detalle de lo que ocurrió dentro; para eso siguen haciendo falta logs (idealmente con el mismo `traceId`).
- **No funciona automáticamente entre servicios sin propagación** — si un servicio no reenvía las cabeceras de contexto al llamar al siguiente, la traza se corta ahí y aparecen como peticiones sin relación.
- **No es gratis en volumen** — trazar el 100% del tráfico en un sistema con mucha carga tiene un coste de almacenamiento y red considerable; por eso existe el *sampling*.
- **No diagnostica el problema por ti** — te enseña dónde se ha ido el tiempo, pero interpretar por qué ese tramo es lento sigue siendo trabajo humano.

## Buenas prácticas avanzadas

- **Propaga el contexto en toda comunicación entre servicios, no solo en HTTP** — cuando un servicio publica un mensaje a una cola o un bus (RabbitMQ, Azure Service Bus), hay que propagar el `traceId` en los metadatos del mensaje; si no, la traza se corta en el momento exacto en que más interesaría seguirla: el salto asíncrono entre sistemas.
- **Añade spans manuales en los pasos de negocio que importan, no solo en las llamadas de red** — la instrumentación automática cubre HTTP y acceso a datos, pero un paso como "calcular el precio con descuentos aplicados" es invisible salvo que crees un span explícito para él; sin eso, un problema de lógica de negocio lenta se confunde con "la base de datos va lenta".
- **Aplica *sampling* con reglas, no al azar uniforme** — descartar trazas al azar es fácil de implementar pero puede hacerte perder justo la traza del error raro que buscabas. Un enfoque mejor conserva siempre las trazas con error o con latencia por encima de un umbral, y muestrea con menos frecuencia el tráfico "normal" y sano.
- **Revisa que los relojes de los servicios estén sincronizados** — una traza reconstruye el orden de los spans a partir de sus marcas de tiempo; si los relojes de dos servicios están desincronizados aunque sea por unos segundos, la traza puede mostrar un span "empezando antes" de que termine su padre, lo que confunde más de lo que ayuda.

---

*En resumen: el tracing distribuido pone un mismo hilo conductor (el traceId) a una petición que cruza varios servicios, para poder ver de un vistazo por dónde ha pasado y en qué tramo se ha ido el tiempo.*
