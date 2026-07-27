# Tracing Distribuido

## ¿Qué es?

El *tracing* distribuido permite seguir el recorrido completo de **una única petición** a través de todos los servicios que la atienden, registrando cuánto tarda cada tramo. El recorrido entero se llama **traza** (*trace*) y cada tramo es un **span**.

## ¿Por qué existe?

En una aplicación única, averiguar por qué una petición tarda dos segundos es cuestión de mirar su log o de perfilarla en local. En un sistema distribuido —una API de pedidos que llama a pagos, que a su vez llama a una pasarela externa— la misma pregunta obliga a reconstruir a mano qué pasó en tres sistemas con logs separados y nada que los conecte.

Y no es solo incómodo: es **inviable**. Los logs de los tres servicios están intercalados con los de otras mil peticiones simultáneas. Sin un identificador común, distinguir cuáles pertenecen a la petición que investigas consiste en cruzar marcas de tiempo y esperar acertar.

El tracing lo resuelve asignando un identificador único a la petición que **viaja con ella** de servicio en servicio. Cada servicio añade su tramo con lo que hizo y cuánto tardó, y al final el recorrido se reconstruye como si fuera uno solo.

> Piensa en el número de seguimiento de un paquete: cada almacén por el que pasa añade un sello con la hora, pero es el mismo número el que permite ver el trayecto completo, aunque lo hayan tocado empresas distintas.

## ¿Cuándo y para qué se usa?

En cuanto una petición cruza más de un proceso. Los casos concretos:

- **Localizar el cuello de botella.** Confirmar un pedido tarda tres segundos: ¿es la base de datos, la pasarela de pago o el servicio de inventario?
- **Descubrir dependencias que nadie recuerda.** Un endpoint que acaba llamando a un servicio de terceros porque alguien lo añadió hace dos años.
- **Entender un error en contexto.** Ver no solo la excepción, sino todo lo que pasó antes en esa misma petición y en los otros servicios.

Y también fuera de HTTP: un mensaje que viaja por una cola merece exactamente el mismo seguimiento, aunque ahí cuesta más trabajo (lo vemos abajo).

---

## Traza, span y el árbol

Una **traza** es el conjunto de todo lo que ocurrió al atender una petición. Un **span** es una unidad de trabajo con nombre, hora de inicio y duración. Los spans forman un árbol: cada uno conoce a su padre.

```
Trace 4bf92f3577b34da6a3ce929d0e0e4736                            2 340 ms
│
├─ [pedidos]     POST /api/pedidos/4711/confirmar                 2 340 ms
│  ├─ [pedidos]  SELECT pedidos WHERE id = 4711                       12 ms
│  ├─ [pagos]    POST /api/cobros                                  2 180 ms
│  │  └─ [pagos] HTTP POST api.pasarela-pago.com/v1/charges        2 160 ms   ← aquí
│  ├─ [inventario] POST /api/reservas                                 95 ms
│  └─ [pedidos]  UPDATE pedidos SET estado = 'Confirmado'              18 ms
```

Esa vista se lee en dos segundos y contesta la pregunta: el problema no es tu código, es la pasarela externa. Sin trazas, la misma conclusión sale de cruzar los logs de tres servicios.

Fíjate en la relación entre los tiempos: el span padre (2 340 ms) **incluye** a sus hijos. Cuando la suma de los hijos es mucho menor que el padre, el tiempo se está yendo en el propio servicio y no en sus llamadas — que es una pista distinta y también útil.

## El contexto de traza

Lo que hace posible el árbol son tres identificadores que acompañan a cada span:

| Campo | Qué es |
|---|---|
| `traceId` | 16 bytes. **El mismo para toda la traza**, en todos los servicios. |
| `spanId` | 8 bytes. Identifica este tramo concreto. |
| `parentSpanId` | El `spanId` de quien lo originó. Vacío en el primero. |

Ese conjunto se llama **contexto de traza**, y propagarlo de un servicio a otro es todo el mecanismo.

### La cabecera `traceparent`

El formato está estandarizado por el W3C, lo que permite que servicios escritos en lenguajes distintos y con herramientas distintas participen en la misma traza:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ─┬ ────────────────┬─────────────── ────────┬─────── ─┬
              │                 │                        │         │
              │                 │                        │         └─ flags: 01 = muestreada
              │                 │                        └─────────── spanId del emisor
              │                 └──────────────────────────────────── traceId
              └────────────────────────────────────────────────────── versión del formato
```

El servicio que recibe esa cabecera sabe que su span pertenece a la traza `4bf92f35...` y que su padre es `00f067aa...`. Con eso reconstruye el árbol sin coordinación previa.

El último byte importa más de lo que parece: si vale `00`, la traza **no está muestreada** y los servicios de aguas abajo pueden descartar sus spans. Es lo que permite que la decisión de muestreo se tome una vez, al principio, y la respeten todos.

## `Activity`: los spans en .NET

.NET no usa la palabra "span": su clase equivalente se llama `Activity` y vive en `System.Diagnostics`, dentro del propio runtime. Es anterior a OpenTelemetry y hoy es compatible con él, así que **no hace falta ninguna librería para crear spans**.

Se empieza declarando un `ActivitySource`, normalmente uno por servicio o por módulo:

```csharp
public static class Telemetria
{
    public const string NombreFuente = "MiTienda.Pedidos";
    public static readonly ActivitySource Fuente = new(NombreFuente);
}
```

Y se crean actividades a partir de él:

```csharp
using var activity = Telemetria.Fuente.StartActivity("ConfirmarPedido");
activity?.SetTag("pedido.id", pedido.Id);
activity?.SetTag("pedido.total", pedido.Total);

await ProcesarAsync(pedido);
```

Tres detalles imprescindibles de ese fragmento:

- **El `?.` no es opcional.** `StartActivity` devuelve `null` si nadie está escuchando esa fuente, que es el caso por defecto. Sin el operador condicional, la aplicación revienta en cuanto se ejecuta sin telemetría configurada — por ejemplo, en los tests.
- **El `using` cierra el span.** La duración se calcula al liberar la actividad. Sin `using`, el span queda abierto y no aparece o aparece con una duración absurda.
- **`StartActivity` enlaza sola con el padre.** Toma `Activity.Current` como padre automáticamente, y `Activity.Current` sigue el flujo asíncrono. No hay que pasar nada a mano entre métodos.

### Registrar el resultado

Un span que termina sin más se considera correcto. Cuando algo falla hay que decirlo, o la traza mostrará en verde una petición que reventó:

```csharp
try
{
    await pasarela.CobrarAsync(pedido);
}
catch (Exception ex)
{
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
    activity?.AddException(ex);
    throw;
}
```

`SetStatus` marca el span en rojo en el visor y —esto es lo importante— permite filtrar "enséñame las trazas con error", que es la consulta que más se usa. `AddException` adjunta el tipo, el mensaje y el *stack trace* como evento dentro del span.

### `ActivityKind`

Indica el papel del span en la comunicación, y es lo que permite al visor dibujar bien las relaciones entre servicios:

| Kind | Cuándo |
|---|---|
| `Server` | Atiendes una petición entrante |
| `Client` | Llamas a otro servicio |
| `Producer` | Publicas un mensaje en una cola |
| `Consumer` | Consumes un mensaje de una cola |
| `Internal` | Trabajo interno. El valor por defecto. |

```csharp
using var activity = Telemetria.Fuente.StartActivity("PublicarPedidoConfirmado", ActivityKind.Producer);
```

Dejarlo todo en `Internal` funciona, pero el visor pierde la capacidad de distinguir "esto es una llamada saliente" de "esto es trabajo local", que es justo lo que se mira al buscar latencia de red.

## Automático frente a manual

La instrumentación automática cubre las fronteras técnicas sin que escribas nada. Con [OpenTelemetry](OpenTelemetry.md):

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()    // peticiones HTTP entrantes
        .AddHttpClientInstrumentation()    // llamadas HTTP salientes
        .AddEntityFrameworkCoreInstrumentation()
        .AddSource(Telemetria.NombreFuente));   // ← tus spans manuales
```

Ese `AddSource` con el nombre **exacto** del `ActivitySource` es lo que se olvida siempre: sin él, la instrumentación automática funciona y tus spans propios no aparecen, lo que hace pensar que el código está mal.

Lo que da gratis es mucho: cada petición, cada llamada saliente y cada consulta a base de datos generan su span con sus atributos estándar. Con eso solo, ya tienes el árbol del ejemplo del principio.

Lo que **no** puede saber es qué partes de tu lógica importan. Este span no existe salvo que lo crees:

```csharp
using var activity = Telemetria.Fuente.StartActivity("CalcularDescuentos");
activity?.SetTag("promociones.evaluadas", promociones.Count);

var total = calculadora.Aplicar(pedido, promociones);
```

Sin él, si el cálculo de descuentos tarda 800 ms, esos 800 ms aparecen como tiempo sin explicar dentro del span de la petición, y la conclusión natural —equivocada— es que "la aplicación va lenta". La regla práctica: crea un span manual para cada paso de negocio que pueda tardar y que quieras poder señalar con el dedo.

## Propagar el contexto

En HTTP esto funciona solo. `HttpClient` inyecta la cabecera `traceparent` en cada petición saliente y ASP.NET Core la lee en cada entrante. No hay que hacer nada.

**En las colas de mensajes, no.** Y aquí está el punto donde la mayoría de los sistemas pierden la traza, precisamente en el salto que más interesaría seguir.

Un mensaje publicado en una cola no lleva cabeceras HTTP. Si nadie propaga el contexto, el consumidor arranca una traza nueva y el recorrido queda partido en dos mitades sin relación aparente.

Al publicar, hay que meter el contexto en los metadatos del mensaje:

```csharp
using var activity = Telemetria.Fuente.StartActivity("PublicarPedidoConfirmado", ActivityKind.Producer);

var mensaje = new ServiceBusMessage(cuerpo);

// Inyectar el contexto actual en las propiedades del mensaje
Propagators.DefaultTextMapPropagator.Inject(
    new PropagationContext(activity?.Context ?? default, Baggage.Current),
    mensaje.ApplicationProperties,
    (props, clave, valor) => props[clave] = valor);

await sender.SendMessageAsync(mensaje);
```

Y al consumir, extraerlo y usarlo como padre:

```csharp
var contexto = Propagators.DefaultTextMapPropagator.Extract(
    default,
    args.Message.ApplicationProperties,
    (props, clave) => props.TryGetValue(clave, out var v) ? [v?.ToString()] : []);

using var activity = Telemetria.Fuente.StartActivity(
    "ProcesarPedidoConfirmado",
    ActivityKind.Consumer,
    contexto.ActivityContext);       // ← el padre viene del mensaje
```

Ese tercer argumento es todo el truco: enlaza el span del consumidor con la traza que originó el mensaje, aunque hayan pasado cinco minutos y sea otro proceso en otra máquina.

Merece la pena hacerlo una vez, envuelto en un middleware o un decorador del publicador y del consumidor, en lugar de repetirlo en cada sitio. El contexto de mensajería está en [Mensajería Asíncrona](../mensajeria-asincrona/README.md).

## Muestreo

Guardar el 100 % de las trazas de un sistema con tráfico real es inviable: el volumen de datos supera con creces al de los logs. El muestreo (*sampling*) decide cuáles se conservan, y **cómo** se decide marca la diferencia.

**Head sampling** decide al empezar la traza, antes de saber qué va a pasar:

```csharp
.WithTracing(t => t.SetSampler(new TraceIdRatioBasedSampler(0.1)))   // el 10 %
```

Es barato y tiene un defecto grave: descarta al azar, así que la traza del error raro que buscabas tiene un 90 % de probabilidades de no existir. Justo la que necesitabas.

**Tail sampling** decide **después** de completar la traza, cuando ya se sabe si hubo error y cuánto tardó. La política habitual:

```
conservar SIEMPRE  si la traza tiene algún span con error
conservar SIEMPRE  si la duración supera 1 s
conservar el 5 %   del resto
```

Es lo que quieres, y no se hace en la aplicación: se configura en el [OpenTelemetry Collector](OpenTelemetry.md), que es quien puede ver la traza entera antes de decidir.

Un detalle que descoloca: para que el tail sampling funcione, **todos los spans de una traza tienen que llegar al mismo Collector**. Con varias instancias hace falta enrutar por `traceId`, y es la razón principal de que tail sampling sea bastante más caro de montar que head sampling.

## Correlacionar con los logs

La traza dice **dónde** se fue el tiempo; el log dice **qué** pasó. El salto entre ambos es lo que hace útil todo esto, y se consigue metiendo el `traceId` en cada log.

Con OpenTelemetry en .NET, esto ya viene hecho: el proveedor de logs adjunta `TraceId` y `SpanId` automáticamente a todo lo que se registre dentro de una actividad. Con [Serilog](Serilog.md), un enricher hace lo mismo.

El resultado es que desde un span puedes saltar a sus logs:

```
TraceId = '4bf92f3577b34da6a3ce929d0e0e4736'
```

...y desde un log de error, abrir la traza completa que lo rodea. Sin eso, tienes dos herramientas que no se hablan y cada investigación empieza de cero.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| Mis spans manuales no aparecen | Falta `AddSource` con el nombre exacto del `ActivitySource` |
| `NullReferenceException` al instrumentar | Falta el `?.`: `StartActivity` devuelve `null` si nadie escucha |
| Los spans salen con duración cero o rarísima | Falta el `using`: la actividad no se cierra |
| La traza se corta al pasar por una cola | No se propaga el contexto en los metadatos del mensaje |
| Cada servicio genera su propia traza | Alguien no reenvía la cabecera `traceparent` |
| Un span "empieza antes" de que acabe su padre | Relojes de los servidores desincronizados |
| Veo trazas pero no las de los errores | Head sampling aleatorio; hace falta tail sampling |

El de los relojes merece una nota: la traza reconstruye el orden a partir de marcas de tiempo generadas en máquinas distintas. Unos pocos segundos de desfase producen árboles imposibles que confunden más de lo que ayudan. Tener NTP funcionando en todas las máquinas es un requisito previo del tracing, no un detalle de administración.

## Buenas prácticas avanzadas

- **Nombra los spans por la operación, nunca con datos variables dentro.** `StartActivity($"GET /api/pedidos/{id}")` genera un nombre distinto por pedido, y el visor deja de poder agregar: no habrá forma de preguntar "cuánto tarda de media esta operación". El nombre debe ser la plantilla (`GET /api/pedidos/{id}`) y el identificador un atributo (`SetTag("pedido.id", id)`). Es exactamente el mismo problema de cardinalidad que en las [métricas](Metricas.md).
- **Propaga el contexto en cada frontera asíncrona, no solo en HTTP.** Colas, trabajos programados y procesos por lotes cortan la traza por defecto, y son justo los sitios donde reconstruir qué pasó a mano resulta más difícil. Hacerlo una vez en un decorador del publicador y del consumidor cuesta una tarde y evita que la mitad de tu sistema sea invisible.
- **Añade atributos que sirvan para filtrar, no para leer.** Un span con veinte atributos descriptivos es ruido; uno con `pedido.id`, `cliente.tipo` y `metodo_pago` permite responder "¿las trazas lentas son todas de clientes mayoristas?". Los atributos valen por las preguntas que dejan hacer, así que piensa en la consulta antes que en el dato.
- **No instrumentes bucles cerrados ni funciones triviales.** Crear un span por iteración de un bucle de mil elementos genera mil spans que hacen ilegible la traza y disparan el coste, además de añadir sobrecarga real. Instrumenta el bucle entero con un atributo `elementos.procesados`, y baja de nivel solo si esa medición señala un problema.
- **Comprueba que la traza sobrevive a un despliegue parcial.** Durante una actualización progresiva conviven versiones instrumentadas y no instrumentadas, y un servicio que no propaga el contexto parte todas las trazas que pasen por él. Verificar la propagación en un entorno de preproducción con la mezcla de versiones evita descubrir el agujero justo durante el incidente que querías investigar.

## Recursos didácticos

- [Jaeger en Docker](https://www.jaegertracing.io/docs/latest/getting-started/) — `docker run -p 16686:16686 -p 4317:4317 jaegertracing/all-in-one` levanta un visor completo en un comando. Apuntarle una aplicación .NET propia y ver el primer *waterfall* real es lo que hace que el concepto encaje; ninguna explicación sustituye a ver el árbol de spans de tu propio código.
- [W3C Trace Context](https://www.w3.org/TR/trace-context/) — la especificación de la cabecera `traceparent`, corta y legible. Merece la pena leer la sección del formato una vez para no tratar la cabecera como magia.
- [OpenTelemetry Demo](https://opentelemetry.io/docs/demo/) — una tienda online de microservicios completa, con fallos inyectados a propósito, para investigar trazas reales de un sistema distribuido de verdad sin tener que construirlo.

---

*En resumen: el tracing pone un mismo hilo conductor a una petición que cruza varios servicios — funciona solo en HTTP, hay que propagarlo a mano en las colas, y sin él la latencia de un sistema distribuido es una conjetura.*
