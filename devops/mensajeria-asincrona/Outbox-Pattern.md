# Outbox Pattern

## ¿Qué es?

El *transactional outbox* garantiza que un evento se publique siempre que ocurra el cambio que lo origina: en vez de publicar directamente en el broker, el evento se guarda en una tabla `Outbox` dentro de la **misma transacción** que el dato de negocio, y un proceso aparte lo lee de ahí y lo publica de verdad.

## ¿Por qué existe?

Guardar un pedido en la base de datos y publicar `PedidoConfirmado` en una cola son dos operaciones contra **dos sistemas distintos**. No se pueden envolver juntas en una transacción, y eso deja un hueco que no se puede tapar reordenando el código.

Mira las dos opciones ingenuas. Publicar primero:

```csharp
// ❌ Si el guardado falla, facturación ya reaccionó a un pedido que no existe
await bus.PublishAsync(new PedidoConfirmado { PedidoId = pedido.Id });
await repositorio.GuardarAsync(pedido);
```

O guardar primero:

```csharp
// ❌ Si el proceso muere aquí en medio, el pedido existe y nadie se entera jamás
await repositorio.GuardarAsync(pedido);
await bus.PublishAsync(new PedidoConfirmado { PedidoId = pedido.Id });
```

Esto se llama el **problema de la doble escritura**, y no tiene solución con dos recursos independientes. Envolverlo en un `try/catch` no vale: si el proceso muere entre las dos líneas —un reinicio, un despliegue, el *OOM killer*— no hay `catch` que se ejecute.

El outbox lo resuelve reduciéndolo a **un solo recurso transaccional**. El evento se guarda como una fila más, en la misma transacción que el dato: o se guardan los dos o ninguno. Publicarlo pasa a ser un segundo paso, separado y reintentable, que ya no puede comprometer la consistencia del primero.

> Piensa en un contrato importante: al firmarlo, archivas una copia en la carpeta del caso en el mismo acto —eso no puede fallar por separado—. Enviárselo al notario es un paso posterior, que se puede reintentar si el mensajero no lo consigue hoy, sin que eso ponga en duda que el contrato se firmó.

## ¿Cuándo y para qué se usa?

Cuando un evento cruza un límite de proceso y dispara efectos que no se pueden deshacer: cobrar una tarjeta, generar una factura, enviar un email que ya no se puede "desenviar". Si perder el evento —o tener un evento fantasma de algo que no ocurrió— es un problema real de negocio, el outbox es la respuesta estándar.

No hace falta para eventos que se quedan dentro del mismo proceso y no necesitan sobrevivir a un fallo entre guardar y publicar: para eso están los [Domain Events](../../arquitectura-de-software/patrones-de-diseno/Domain-Events.md) en memoria.

---

## La tabla

```sql
CREATE TABLE OutboxMessages (
    Id            UNIQUEIDENTIFIER PRIMARY KEY,
    Tipo          NVARCHAR(200)    NOT NULL,
    Payload       NVARCHAR(MAX)    NOT NULL,
    CreadoEn      DATETIME2        NOT NULL,
    ProcesadoEn   DATETIME2        NULL,
    Intentos      INT              NOT NULL DEFAULT 0,
    UltimoError   NVARCHAR(MAX)    NULL
);

CREATE INDEX IX_Outbox_Pendientes
    ON OutboxMessages (CreadoEn)
    WHERE ProcesadoEn IS NULL;
```

Las decisiones de este esquema:

- **`Tipo` + `Payload`** en lugar de una tabla por evento. Una sola tabla con el nombre del tipo y el JSON serializado sirve para todos los eventos y es mucho más fácil de mantener.
- **`ProcesadoEn` nulable** en vez de borrar la fila al publicar. Conserva el rastro para depurar y permite archivar en bloque después.
- **`Intentos` y `UltimoError`** para detectar el mensaje que falla siempre y no dejar que bloquee la cola.
- **El índice filtrado** es la pieza de rendimiento clave. El publicador consulta *"dame lo no procesado"* cada pocos segundos, para siempre; sin un índice que cubra solo las filas pendientes, esa consulta acaba recorriendo millones de filas ya publicadas.

## Guardar el dato y el evento juntos

Este es el corazón del patrón: una única transacción.

```csharp
await using var transaccion = await conexion.BeginTransactionAsync();

await repositorioPedidos.GuardarAsync(pedido, transaccion);

await outbox.AgregarAsync(new OutboxMessage(
    Tipo: nameof(PedidoConfirmado),
    Payload: JsonSerializer.Serialize(new PedidoConfirmado { PedidoId = pedido.Id })),
    transaccion);

await transaccion.CommitAsync();
// o se guardan los dos, o ninguno: no hay hueco posible entre ambos
```

Con Entity Framework Core es aún más directo, porque `SaveChangesAsync` ya envuelve todo en una transacción:

```csharp
contexto.Pedidos.Add(pedido);
contexto.OutboxMessages.Add(new OutboxMessage(
    nameof(PedidoConfirmado),
    JsonSerializer.Serialize(new PedidoConfirmado { PedidoId = pedido.Id })));

await contexto.SaveChangesAsync();   // una sola transacción para las dos tablas
```

El requisito imprescindible: **la tabla `Outbox` tiene que estar en la misma base de datos** que el dato de negocio. Si está en otra, vuelves a tener dos recursos y el problema original intacto.

## El publicador

Un proceso en segundo plano lee lo pendiente y lo publica. En .NET, un `BackgroundService`:

```csharp
public class PublicadorOutbox(IServiceProvider sp, ILogger<PublicadorOutbox> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                await PublicarLoteAsync(ct);
            }
            catch (Exception ex)
            {
                log.LogError(ex, "Fallo en el ciclo del outbox");
            }

            await Task.Delay(TimeSpan.FromSeconds(5), ct);
        }
    }
}
```

El `try/catch` que envuelve **todo** el ciclo no es defensivo de más: sin él, una excepción no controlada mata el `BackgroundService` en silencio y la aplicación sigue funcionando con el outbox parado. Los eventos se acumulan en la tabla sin que nadie lo note.

Y el cuerpo del ciclo:

```csharp
private async Task PublicarLoteAsync(CancellationToken ct)
{
    var pendientes = await outbox.ObtenerPendientesAsync(lote: 50, ct);

    foreach (var mensaje in pendientes)
    {
        try
        {
            await bus.PublishAsync(mensaje.Tipo, mensaje.Payload, ct);
            await outbox.MarcarProcesadoAsync(mensaje.Id, ct);
        }
        catch (Exception ex)
        {
            await outbox.RegistrarFalloAsync(mensaje.Id, ex.Message, ct);
        }
    }
}
```

Fíjate en el orden: **primero se publica y después se marca**. Al revés perderías mensajes si el proceso muere entre las dos operaciones. Así, en el peor caso, se publica dos veces — lo cual es aceptable y es justo el motivo por el que el consumidor debe ser idempotente.

Fíjate también en que el `catch` es **por mensaje**, no por lote. Si el tercero de cincuenta falla, los otros cuarenta y nueve siguen adelante.

## Que dos instancias no publiquen lo mismo

En cuanto la aplicación corre en dos réplicas —lo normal— las dos ejecutan el publicador y las dos leen las mismas filas pendientes. Cada evento se publica el doble.

La solución no es coordinar los procesos, sino dejar que la base de datos reparta el trabajo. En SQL Server:

```sql
UPDATE TOP (50) OutboxMessages WITH (READPAST, UPDLOCK)
SET ProcesadoEn = SYSUTCDATETIME()
OUTPUT inserted.Id, inserted.Tipo, inserted.Payload
WHERE ProcesadoEn IS NULL;
```

Las dos pistas hacen el trabajo: **`UPDLOCK`** bloquea las filas seleccionadas y **`READPAST`** hace que otra sesión **salte** las bloqueadas en lugar de esperarlas. Dos instancias ejecutando esto a la vez se llevan lotes distintos sin bloquearse.

El equivalente en PostgreSQL:

```sql
SELECT id, tipo, payload FROM outbox_messages
WHERE procesado_en IS NULL
ORDER BY creado_en
LIMIT 50
FOR UPDATE SKIP LOCKED;
```

`FOR UPDATE SKIP LOCKED` es exactamente la misma idea y es la razón por la que este patrón funciona bien sobre una base de datos relacional cualquiera.

Hay una diferencia importante entre las dos variantes. La de SQL Server marca como procesado **antes** de publicar: si la publicación falla, ese evento se pierde. Es más simple y solo vale si puedes tolerarlo. Lo habitual es una columna intermedia (`BloqueadoHasta`) que reserva la fila temporalmente y la libera si nadie la confirma, de modo que un proceso caído no se lleve el mensaje consigo.

## El mensaje que falla siempre

Si un evento no se puede publicar —un payload corrupto, un tipo que ya no existe— el publicador lo reintentará en cada ciclo, para siempre, y si además respetas el orden, bloqueará todo lo que venga detrás.

Para eso está la columna `Intentos`:

```csharp
var pendientes = await outbox.ObtenerPendientesAsync(lote: 50, maxIntentos: 10, ct);
```

Al superar el límite, la fila deja de seleccionarse y queda a la espera de revisión manual. Es el mismo concepto que una [dead-letter queue](Dead-Letter-Queues.md), aplicado a la tabla en vez de a la cola — y necesita lo mismo: **una alerta**. Una consulta periódica de "cuántas filas llevan más de N intentos" o "hace cuánto se creó el pendiente más antiguo" es lo que evita que el outbox se atasque en silencio durante una semana.

## Si el orden importa

Por defecto, publicar el lote en paralelo es más rápido y no garantiza ningún orden. Si dos eventos de un mismo pedido deben llegar en orden, hay que forzarlo:

```csharp
var porAgregado = pendientes.GroupBy(m => m.AgregadoId);

await Parallel.ForEachAsync(porAgregado, ct, async (grupo, token) =>
{
    foreach (var mensaje in grupo.OrderBy(m => m.CreadoEn))   // en serie dentro del grupo
    {
        await bus.PublishAsync(mensaje.Tipo, mensaje.Payload, token);
        await outbox.MarcarProcesadoAsync(mensaje.Id, token);
    }
});
```

Se serializa dentro de cada agregado y se paraleliza entre agregados distintos: se conserva el orden donde importa sin sacrificar todo el rendimiento. Requiere guardar el `AgregadoId` como columna, y encaja de forma natural con el `SessionId` de [Azure Service Bus](Azure-Service-Bus.md).

## Limpiar la tabla

Una tabla `Outbox` que nunca se limpia crece hasta que las consultas se degradan. Una tarea periódica —vía [cron](../despliegue-en-vps/Tareas-Programadas-con-Cron.md) o como otro `BackgroundService`— con borrado por lotes:

```sql
DELETE TOP (5000) FROM OutboxMessages
WHERE ProcesadoEn IS NOT NULL
  AND ProcesadoEn < DATEADD(day, -7, SYSUTCDATETIME());
```

El `TOP (5000)` no es cosmético: un `DELETE` de millones de filas de golpe escala el bloqueo a toda la tabla y detiene las inserciones de pedidos mientras dure. Borrar por lotes, repitiendo hasta que no queden, mantiene la tabla operativa durante la limpieza.

## CDC como alternativa al sondeo

El publicador de arriba hace *polling*: consulta cada cinco segundos aunque no haya nada. Funciona bien y es fácil de entender, pero añade latencia y carga constante.

La alternativa es **Change Data Capture**: leer el registro de transacciones de la base de datos y reaccionar a las inserciones sin consultar. Herramientas como Debezium lo hacen y publican directamente en el broker.

| | Polling | CDC |
|---|---|---|
| Latencia | El intervalo (segundos) | Casi inmediata |
| Carga sobre la BD | Una consulta constante | Lee el log, no la tabla |
| Complejidad | Un `BackgroundService` | Una pieza de infraestructura más |

La recomendación práctica: **empieza con polling**. Cinco segundos de retraso son aceptables en casi todos los escenarios de negocio, y CDC añade un componente que hay que desplegar, monitorizar y entender cuando falle. Cambia a CDC cuando el retraso sea un problema medido, no supuesto.

## El reverso: el inbox

El outbox garantiza que el mensaje **sale** exactamente una vez desde el punto de vista del negocio. No garantiza que llegue una sola vez: ya vimos que puede publicarse dos veces.

El patrón simétrico en el consumidor es el **inbox**: una tabla donde se registra el identificador de cada mensaje procesado, comprobada antes de trabajar.

```csharp
if (await inbox.YaProcesadoAsync(mensaje.MessageId))
    return;                                  // duplicado: ignorar

await using var tx = await conexion.BeginTransactionAsync();
await GenerarFacturaAsync(evento.PedidoId, tx);
await inbox.RegistrarAsync(mensaje.MessageId, tx);
await tx.CommitAsync();                      // trabajo y registro, atómicos
```

La clave está en que el trabajo y el registro van en la **misma transacción**. Si fueran dos operaciones separadas, volverías a tener el problema de la doble escritura, esta vez en el lado del consumidor.

Outbox e inbox juntos son lo más parecido a "exactamente una vez" que se puede construir: el emisor no pierde eventos y el receptor no los aplica dos veces.

## Buenas prácticas avanzadas

- **No metas la publicación real dentro de la transacción de negocio.** Es tentador "aprovechar" la transacción abierta para publicar y ahorrarse el publicador. Pero una llamada al broker dentro de una transacción de base de datos mantiene los bloqueos abiertos durante toda la latencia de red, y si el broker va lento, la contención se traslada a la tabla de pedidos. El outbox existe precisamente para que la transacción sea corta y solo toque la base de datos.
- **Guarda el contexto de trazabilidad en la fila.** Añadir columnas con el `TraceId` y el `SpanId` de la petición que originó el evento, y restaurarlos en las cabeceras al publicar, es lo que permite seguir un pedido desde el clic del usuario hasta la factura. Sin eso, la traza se corta en el outbox y depurar un evento perdido consiste en cruzar marcas de tiempo a mano.
- **Serializa el evento en su forma pública, no la entidad de dominio.** Es cómodo hacer `JsonSerializer.Serialize(pedido)` y guardarlo, pero eso convierte tu modelo interno en el contrato público: un renombrado de propiedad rompe a consumidores que no controlas, y además el payload queda con el estado del momento del guardado. Serializa un DTO de evento explícito y versionado.
- **Publica también si el proceso lleva parado un rato, pero con cabeza.** Al arrancar tras una caída larga puede haber miles de eventos pendientes que se publicarán de golpe y saturarán a los consumidores. Un límite de lote y una pequeña pausa entre lotes convierten una avalancha en una recuperación ordenada. Es la diferencia entre recuperarse y provocar un segundo incidente.
- **Prueba el patrón matando el proceso, no con tests unitarios.** Lo que el outbox promete es sobrevivir a un fallo entre dos operaciones, y eso no se verifica con un mock. La prueba real es lanzar carga, matar el proceso a lo bruto en medio y comprobar que ningún pedido quedó sin evento y ningún evento sin pedido. Es la única forma de descubrir que la transacción no abarcaba lo que creías.

## Recursos didácticos

- [Microservices.io — Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) — la ficha canónica del patrón, con el diagrama del problema de la doble escritura y los dos mecanismos de publicación (polling y CDC) contrastados.
- [Debezium](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html) — la implementación de CDC más extendida, con un enrutador específico para outbox. Aunque no vayas a usarlo, su documentación explica muy bien qué aporta CDC frente al sondeo.
- [Enterprise Integration Patterns — Idempotent Receiver](https://www.enterpriseintegrationpatterns.com/patterns/messaging/IdempotentReceiver.html) — la otra mitad del problema: por qué el outbox no basta y qué tiene que hacer el consumidor.

---

*En resumen: el outbox guarda el evento en la misma transacción que el dato, para que "guardar" y "avisar" nunca queden a medias — nunca hay un pedido sin su evento, ni un evento de un pedido que no llegó a existir.*
