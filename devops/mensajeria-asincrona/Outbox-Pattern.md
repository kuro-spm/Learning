# Outbox Pattern

## ¿Qué es?

El *transactional outbox* (patrón *outbox*) garantiza que un evento se publique de forma fiable cuando debe ir siempre ligado a un cambio en base de datos: en vez de publicar el evento directamente en el broker, se guarda primero en una tabla `Outbox` dentro de la **misma transacción** que el dato de negocio, y un proceso aparte se encarga después de leer esa tabla y publicarlo de verdad.

## ¿Por qué existe?

Guardar un pedido en base de datos y publicar el evento "pedido confirmado" en una cola son dos operaciones contra dos sistemas distintos (la base de datos y el broker), que no se pueden envolver juntas en una única transacción de forma práctica. Eso deja dos huecos peligrosos: si publicas el evento antes de guardar y el guardado falla, alguien reacciona a un pedido que nunca llegó a existir; si guardas primero y el proceso se cae justo antes de publicar, tienes un pedido confirmado del que nadie se entera nunca.

El patrón *outbox* elimina el hueco usando un solo recurso transaccional: la base de datos. El evento se guarda como una fila más, en la misma transacción que el dato de negocio, así que o se guardan los dos juntos o no se guarda ninguno. Publicarlo de verdad en el broker pasa a ser un segundo paso, separado y reintentable, que ya no compromete la consistencia del primero.

> Piensa en un burofax certificado: al firmar un contrato importante, archivas una copia en la carpeta del caso en el mismo momento de la firma —eso no puede fallar de forma independiente—; el envío del burofax al notario es un paso posterior, que puede reintentarse si el mensajero no lo consigue el primer día, sin que eso ponga en duda que el contrato se firmó.

## ¿Cuándo y para qué se usa?

Cuando un evento debe cruzar un límite (a otro módulo, a otro servicio, a una cola) y ese evento dispara efectos que no se pueden deshacer fácilmente: cobrar una tarjeta, generar una factura, enviar un email que ya no se puede "desenviar". Si perder el evento o tener un evento "fantasma" de algo que no ocurrió es un problema real de negocio, el outbox es la forma estándar de evitarlo. No hace falta para eventos que solo interesan dentro del mismo proceso y no necesitan sobrevivir a un fallo entre guardar y publicar (ver [Domain Events](../../arquitectura-de-software/patrones-de-diseno/Domain-Events.md)).

## Lo mínimo que necesitas saber

**1. Una tabla `Outbox` en la misma base de datos que el dato de negocio**

```sql
CREATE TABLE OutboxMessages (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    Tipo NVARCHAR(200) NOT NULL,
    Payload NVARCHAR(MAX) NOT NULL,
    CreadoEn DATETIME2 NOT NULL,
    ProcesadoEn DATETIME2 NULL
);
```

**2. El dato de negocio y el evento se guardan en la misma transacción**

```csharp
await using var transaccion = await connection.BeginTransactionAsync();

await repositorioPedidos.Guardar(pedido, transaccion);
await outbox.Agregar(new OutboxMessage("PedidoConfirmado", pedido.Id), transaccion);

await transaccion.CommitAsync();
// o los dos se guardan, o ninguno: no hay hueco entre ambos
```

**3. Un proceso aparte lee lo pendiente y lo publica de verdad**

```csharp
var pendientes = await outbox.ObtenerNoProcesados();

foreach (var mensaje in pendientes)
{
    await bus.PublishAsync(mensaje.Tipo, mensaje.Payload);
    await outbox.MarcarComoProcesado(mensaje.Id);
}
```

**4. Ese proceso corre en bucle o reacciona a cambios en la tabla**

Puede ser un *worker* en segundo plano que sondea la tabla cada pocos segundos (*polling*), o un mecanismo de *change data capture* (CDC) que reacciona al log de la propia base de datos sin necesidad de sondear.

## Lo que NO hace

- **No garantiza entrega única, solo entrega segura** — si el proceso se cae entre publicar en el broker y marcar el mensaje como procesado, el mismo mensaje puede reenviarse; el consumidor final sigue necesitando ser idempotente.
- **No sustituye a un broker de mensajería** — es el mecanismo que asegura que el mensaje *sale* de tu aplicación de forma fiable; la entrega, el enrutado y el resto siguen siendo cosa de la cola o el bus (ver [Mensajería Asíncrona](Mensajeria-Asincrona.md)).
- **No es instantáneo** — el evento no sale al broker en el mismo instante que se guarda, sino en el siguiente ciclo del proceso publicador; si la aplicación necesita una respuesta síncrona inmediata, esto no es lo que busca.

## Buenas prácticas avanzadas

- **Limpia o archiva lo ya procesado** — una tabla `Outbox` que crece sin límite porque nunca se borra lo publicado acaba pesando en el rendimiento de las consultas; una tarea periódica que archive o borre mensajes procesados tras una ventana razonable (unos días) evita el problema.
- **No hagas *polling* más agresivo de lo necesario** — sondear la tabla cada pocos milisegundos compite por recursos con la carga normal de la base de datos; ajusta el intervalo al retraso que tu negocio puede tolerar, o usa *change data capture* si necesitas algo casi inmediato sin pagar el coste del sondeo constante.
- **Una sola tabla `Outbox` sirve para varios tipos de evento** — no hace falta una tabla por tipo; una columna `Tipo` más un `Payload` serializado (normalmente JSON) es suficiente y mucho más fácil de mantener que múltiples tablas paralelas.
- **Respeta el orden de creación si el orden importa** — si dos eventos del mismo agregado deben procesarse en el orden en que ocurrieron, el publicador debe respetarlo (normalmente ordenando por fecha de creación) y no procesar el lote en paralelo sin ningún control de orden.

---

*En resumen: el outbox guarda el evento como parte de la misma transacción del dato de negocio, para que "guardar" y "avisar" nunca puedan quedar a medias — nunca hay un pedido sin su evento, ni un evento de un pedido que nunca llegó a existir.*
