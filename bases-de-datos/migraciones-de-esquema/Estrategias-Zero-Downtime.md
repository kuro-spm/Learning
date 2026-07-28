# Estrategias de Migración Zero-Downtime

## ¿Qué es?

Es el conjunto de técnicas para cambiar el esquema de una base de datos en producción sin parar el servicio y sin romper las instancias de la aplicación que todavía ejecutan la versión anterior del código.

## ¿Por qué existe?

Un despliegue con varias instancias **no es instantáneo**. El balanceador va sacando instancias de rotación, reiniciándolas con el código nuevo y devolviéndolas al tráfico, una por una. Durante esos minutos hay dos versiones del código hablando con **el mismo** esquema. Si la migración borró una columna que el código viejo todavía lee, las instancias que aún no se han reiniciado empiezan a devolver errores 500 antes de que el despliegue termine. Estas estrategias existen para que nunca haya un instante en el que el esquema y alguna versión del código que está en producción sean incompatibles.

> Es parecido a cambiar de carril en una autopista con el coche en marcha: no puedes saltar directamente del carril viejo al nuevo, tienes que solaparlos un rato mientras te desplazas de uno a otro sin salirte de la vía en ningún momento.

## ¿Cuándo y para qué se usa?

En cualquier sistema que se despliega sin ventana de mantenimiento. El escenario de esta guía es una tienda online sobre la base de datos `TiendaDB`, con las tablas `Productos`, `Pedidos` y `Clientes`, desplegada en **cuatro instancias detrás de un balanceador**. Los ejemplos de SQL van en dialecto de SQL Server; cuando PostgreSQL hace algo relevante y distinto, se indica.

Trata sobre la **secuencia de cambios**: en qué orden se aplican y por qué. El concepto general de migración, la tabla de control y el debate *forward-only* frente a *rollback* están en [Migraciones de esquema](Migraciones-de-Esquema.md); los comandos concretos de cada herramienta, en [EF Core Migrations](EF-Core-Migrations.md), [Flyway](Flyway.md) y [DbUp](DbUp.md).

---

## La ventana de convivencia

Esto es el fundamento de todo lo demás, así que conviene verlo con reloj. Un despliegue *rolling* de cuatro instancias, con 45 segundos entre el reinicio de una instancia y su *health check* verde:

| Momento | Estado | Instancias con v1 | Instancias con v2 |
|---|---|---|---|
| `t+0:00` | Migración aplicada, esquema nuevo | 4 | 0 |
| `t+0:30` | Instancia 1 reiniciada | 3 | 1 |
| `t+1:15` | Instancia 2 reiniciada | 2 | 2 |
| `t+2:00` | Instancia 3 reiniciada | 1 | 3 |
| `t+2:45` | Instancia 4 reiniciada | 0 | 4 |

El esquema tiene que ser válido para **todas** esas combinaciones intermedias, no solo para la final. Y hay tres cosas que alargan la ventana mucho más de lo que sugiere el cronograma:

- **El drenaje.** Una petición que ya estaba en curso en la instancia 1 sigue ejecutando código v1 unos segundos después de que empiece el reinicio.
- **Los procesos de fondo.** El *worker* que envía los correos de confirmación, la tarea programada que recalcula stock y los consumidores de la cola de mensajes suelen desplegarse aparte, a veces horas después.
- **El *rollback*.** Si el *health check* de la instancia 1 falla, vuelves a 4 × v1 **contra el esquema nuevo**, y te quedas así hasta arreglar el problema. Eso puede ser una tarde entera, no dos minutos.

> **Todo cambio de esquema debe ser compatible hacia atrás con el código que aún está en producción**, incluido el código al que podrías tener que volver.

## Clasificación de los cambios

Esta tabla es el centro de la guía. La pregunta que responde no es "¿esto es lento?" sino "¿puedo aplicarlo mientras el código viejo sigue vivo?".

| Cambio | ¿Directo? | Qué rompe si no | Secuencia segura |
|---|---|---|---|
| Crear una tabla nueva | ✅ | Nada: el código viejo la ignora | Directa |
| Añadir columna *nullable* | ✅ | Nada, salvo que el código viejo haga `SELECT *` mapeado por posición | Directa |
| Añadir columna con `DEFAULT` | ✅ | En SQL Server 2012+ y PostgreSQL 11+ es solo metadato; antes reescribía la tabla | Directa, mirando la versión del motor |
| Crear índice *online* | ✅ | Consume IO y toma un bloqueo breve al empezar y al acabar | `ONLINE = ON` / `CONCURRENTLY` |
| Añadir columna `NOT NULL` sin `DEFAULT` | ❌ | Los `INSERT` del código viejo: `Cannot insert the value NULL into column...` | *Nullable* → *backfill* → `NOT NULL` |
| Borrar una columna | ❌ | `Invalid column name 'Nombre'.` en el código viejo | Dejar de usarla → esperar → borrar |
| Renombrar columna o tabla | ❌ | Igual que borrar: desde fuera no es atómico | Expand / migrate / contract |
| Cambiar el tipo de una columna | ❌ | Reescribe la tabla bajo bloqueo y rompe el mapeo del código viejo | Columna nueva + doble escritura + *backfill* |
| Pasar a `NOT NULL` | ❌ | Valida o reescribe toda la tabla, y rechaza los `INSERT` que no la rellenan | Rellenar primero, endurecer al final |
| Añadir clave ajena | ❌ | Escanea la tabla bajo bloqueo y falla si hay filas huérfanas | `WITH NOCHECK` / `NOT VALID` y validar aparte |
| Añadir restricción `UNIQUE` | ❌ | Falla si ya hay duplicados, y crea un índice que bloquea | Detectar duplicados → limpiar → índice *online* |
| Reducir la longitud de una columna | ❌ | `String or binary data would be truncated...` | `CHECK` previo → limpiar datos → reducir |
| Borrar un índice | ⚠️ | No rompe el esquema, pero una consulta del código viejo puede pasar de 5 ms a 30 s | Comprobar su uso real antes |

Los cambios seguros lo son porque **añaden**. Los peligrosos, porque **quitan o restringen**: el código viejo ya contaba con lo que has quitado.

## Expand / migrate / contract

También se llama *parallel change*: partir un cambio peligroso en tres pasos, cada uno compatible con la versión anterior y la siguiente del código. Lo importante, y lo que casi todas las explicaciones se saltan, es que **son tres despliegues distintos en tres momentos distintos**; si los juntas en uno, has vuelto al punto de partida. Vamos con el renombrado de `Productos.Nombre` a `Productos.NombreProducto`.

**Paso 1 — expand.** Añade lo nuevo sin tocar lo viejo. Se despliega solo, sin cambios de código:

```sql
ALTER TABLE Productos ADD NombreProducto NVARCHAR(200) NULL;
```

Metadato puro: no reescribe la tabla y termina en milisegundos. Después de esto la columna existe y está a `NULL` en los 10 millones de filas. El código v1 no la conoce y sigue funcionando igual.

**Paso 2 — migrate.** Aquí va el trabajo de verdad. El código nuevo escribe en **las dos** columnas y sigue leyendo de la vieja:

```csharp
// Producto: durante la ventana, las dos columnas se mantienen sincronizadas
public void CambiarNombre(string nombre)
{
    Nombre = nombre;           // la columna que lee el código v1
    NombreProducto = nombre;   // la columna que leerá el código final
}
```

Con la doble escritura activa ya no entran filas nuevas con `NombreProducto` a `NULL`, y solo entonces tiene sentido el *backfill* de las filas antiguas. Cuando el *backfill* acaba, otro despliegue cambia la lectura:

```csharp
// ❌ Lee solo la nueva: devuelve null en toda fila que el backfill no haya tocado
var etiqueta = producto.NombreProducto;

// ✅ Lee la nueva y cae a la vieja mientras la transición no ha terminado
var etiqueta = producto.NombreProducto ?? producto.Nombre;
```

**El orden de lectura importa** porque decide qué pasa con una fila a medio migrar. Leer la vieja es seguro siempre, pero te ata a ella. Leer la nueva es lo que quieres al final, pero solo es correcto cuando el *backfill* ha terminado *y* la doble escritura está desplegada en las cuatro instancias. El `??` es el puente entre las dos situaciones, y hace que el paso sea seguro incluso si el *backfill* se quedó a medias.

**Paso 3 — contract.** Cuando ninguna instancia lee ni escribe `Nombre`, y solo entonces:

```sql
ALTER TABLE Productos DROP COLUMN Nombre;
```

Si comprimes esto en un solo despliegue, durante la ventana de convivencia habrá instancias v1 pidiendo `Nombre` a una tabla que ya no la tiene, y los logs se llenan de `Msg 207, Level 16, State 1 — Invalid column name 'Nombre'.` en la mitad de las peticiones.

## Pasar una columna a `NOT NULL`

Añadir una columna obligatoria de golpe rompe cualquier `INSERT` del código que aún no la conoce. La secuencia es siempre: *nullable* primero, rellenar, endurecer al final.

```sql
ALTER TABLE Pedidos ADD NumeroSeguimiento NVARCHAR(50) NULL;   -- 1: nullable
-- 2: backfill por lotes de las filas existentes (siguiente sección)
-- 3: solo cuando todas las instancias la rellenan siempre
ALTER TABLE Pedidos ALTER COLUMN NumeroSeguimiento NVARCHAR(50) NOT NULL;
```

Lo que la versión corta de esta receta no dice es que **el paso 3 no es gratis**. `ALTER COLUMN ... NOT NULL` tiene que garantizar que ninguna fila viola la restricción, así que valida la tabla entera —y si además cambia el tipo o la longitud, la reescribe— manteniendo un bloqueo exclusivo de esquema mientras dura. Sobre `Pedidos` con 30 millones de filas eso son minutos, no milisegundos: se mide antes con volumen realista y se aplica con la protección de la sección de la cola de bloqueos.

En **PostgreSQL** hay una vuelta elegante. Desde la versión 12, si ya existe un `CHECK` validado que garantiza lo mismo, `SET NOT NULL` lo aprovecha y se salta el escaneo:

```sql
ALTER TABLE pedidos ADD CONSTRAINT chk_seguimiento
    CHECK (numero_seguimiento IS NOT NULL) NOT VALID;   -- instantáneo
ALTER TABLE pedidos VALIDATE CONSTRAINT chk_seguimiento; -- escanea sin bloquear
ALTER TABLE pedidos ALTER COLUMN numero_seguimiento SET NOT NULL;  -- metadato
ALTER TABLE pedidos DROP CONSTRAINT chk_seguimiento;
```

## Cambiar el tipo de una columna

Un `ALTER COLUMN` que cambia el tipo no se hace en sitio por dos motivos independientes, y basta cualquiera de los dos: el motor reescribe fila a fila la tabla completa bajo bloqueo, y el código v1 sigue mapeando la columna al tipo antiguo, así que un `decimal` que pasa a `int` le llega truncado o le rompe la deserialización.

La secuencia es la misma canción con otra letra —columna nueva, doble escritura, *backfill*, cambio de lectura, borrado—, y arranca con una sola línea:

```sql
ALTER TABLE Pedidos ADD TotalDecimal DECIMAL(10,2) NULL;
```

A partir de ahí, la doble escritura mantiene `Total` (el `float` que lee v1) y `TotalDecimal` sincronizados, con la conversión escrita en un único sitio del código. Un aviso específico de este caso: si el tipo nuevo es más estrecho, el *backfill* fallará **a mitad de camino**. Cuenta antes cuántas filas no caben con un `SELECT COUNT(*) FROM Pedidos WHERE Total > 99999999.99`. Si no devuelve `0`, el problema no es la migración: es que tienes datos que el modelo nuevo no admite, y hay que decidir qué hacer con ellos antes de tocar el esquema.

## El *backfill* por lotes, con el cálculo

Aquí es donde se pierden los despliegues. Este `UPDATE` parece inofensivo:

```sql
-- ❌ Una sola transacción sobre 10 millones de filas
UPDATE Productos SET NombreProducto = Nombre;
```

Lo que ocurre de verdad, con `Productos` a 10 millones de filas y `Nombre` promediando 40 caracteres:

- **Los bloqueos escalan.** SQL Server empieza tomando bloqueos de fila, pero al pasar de unos 5 000 en la misma tabla los cambia por **un bloqueo exclusivo de la tabla entera**. A partir de ahí nadie puede leer ni escribir en `Productos`: el catálogo de la tienda deja de responder.
- **El log de transacciones crece y no se recicla.** Cada fila genera un registro de log, y **nada** puede reciclarse hasta el `COMMIT` porque la transacción sigue abierta. A ~200 bytes por registro, 10 millones de filas son unos **2 GB de log**. Si el fichero estaba dimensionado a 500 MB, tiene que crecer cuatro veces en caliente; y si el disco no da, la transacción entera se deshace después de haber estado bloqueando diez minutos.
- **No es interrumpible.** A los ocho minutos, si alguien lo cancela, el `ROLLBACK` tarda otro tanto y no has avanzado nada.

La versión por lotes hace el mismo trabajo sin ninguno de los tres problemas:

```sql
DECLARE @ultimoId BIGINT = 0, @filas INT = 1;
DECLARE @lote TABLE (Id BIGINT);

WHILE @filas > 0
BEGIN
    DELETE FROM @lote;

    ;WITH Lote AS (
        SELECT TOP (5000) Id, Nombre, NombreProducto
        FROM Productos WHERE Id > @ultimoId ORDER BY Id
    )
    UPDATE Lote SET NombreProducto = Nombre
    OUTPUT inserted.Id INTO @lote;

    SET @filas = @@ROWCOUNT;
    SELECT @ultimoId = ISNULL(MAX(Id), @ultimoId) FROM @lote;

    WAITFOR DELAY '00:00:00.100';   -- deja respirar al resto del sistema
END
```

Dos mil iteraciones de unos 40 ms cada una, más la pausa: unos **cuatro minutos de reloj**, pero sin un solo instante de tabla bloqueada. Los tres beneficios concretos:

1. **Bloqueos cortos.** Cada lote toma sus bloqueos y los suelta al terminar; las consultas del catálogo esperan decenas de milisegundos, no minutos. Matiz de experto: el umbral de escalada está justo en ~5 000 bloqueos, así que 5 000 filas es el borde. Si la tabla tiene varios índices y cada fila consume más de un bloqueo, baja el lote a 2 000.
2. **El log se recicla.** Cada lote es su propia transacción: al confirmar, ese espacio de log queda reutilizable en modo `SIMPLE`, o en cuanto pase la siguiente copia de log en modo `FULL`. El log no crece más de lo que ocupa un lote.
3. **Es interrumpible y reanudable.** El `WHERE Id > @ultimoId` avanza por clave primaria. Si paras el proceso, lo copiado está confirmado: guarda `@ultimoId` y retoma desde ahí, sin `ROLLBACK` gigante que esperar.

En **PostgreSQL** cambia el motivo pero no la conclusión: no hay escalada de bloqueos, pero MVCC crea una **versión muerta por cada fila actualizada**. Un `UPDATE` de 10 millones de filas duplica el tamaño físico de la tabla de golpe y deja al autovacuum un trabajo enorme. Por lotes, el autovacuum va limpiando entre uno y otro.

## Índices sin bloquear la tabla

En SQL Server la variante *online* permite crear el índice mientras la tabla sigue aceptando lecturas y escrituras:

```sql
CREATE INDEX IX_Pedidos_ClienteId ON Pedidos(ClienteId) WITH (ONLINE = ON);
```

Dos avisos. El primero: **`ONLINE = ON` es una característica de la edición Enterprise**; en Standard el comando falla, y si lo quitas la creación toma un bloqueo compartido que impide todas las escrituras durante lo que tarde. El segundo, que es el que sorprende: incluso con `ONLINE = ON` hace falta un bloqueo exclusivo de esquema **breve al empezar y al acabar**, así que sigue expuesto a la cola de bloqueos de la sección siguiente. Si el motor lo soporta, añadir `RESUMABLE = ON, MAX_DURATION = 30` (que exige `ONLINE = ON`) permite pausar y retomar en lugar de tirar media hora de trabajo.

En **PostgreSQL** el equivalente es `CONCURRENTLY`, con una peculiaridad importante:

```sql
CREATE INDEX CONCURRENTLY idx_pedidos_cliente ON pedidos (cliente_id);
```

**No puede ejecutarse dentro de una transacción.** Esto choca de frente con casi todas las herramientas de migración, que envuelven cada script en un `BEGIN`/`COMMIT`; hay que marcar explícitamente ese script como "fuera de transacción". Y si el comando falla o lo cancelas, **deja el índice a medias en estado inválido**: ocupa espacio, penaliza las escrituras y el planificador no lo usa. Hay que borrarlo a mano antes de reintentar. Esta consulta los detecta:

```sql
SELECT t.relname AS tabla, c.relname AS indice FROM pg_index i
JOIN pg_class c ON c.oid = i.indexrelid  JOIN pg_class t ON t.oid = i.indrelid
WHERE i.indisvalid = false;
```

Si devuelve `pedidos | idx_pedidos_cliente`, se limpia con `DROP INDEX CONCURRENTLY idx_pedidos_cliente;`, que tampoco bloquea.

## La cola de bloqueos

Este es el efecto más contraintuitivo del tema, y explica los incidentes en los que "solo añadimos una columna" y se cayó la tienda entera. Un `ALTER TABLE`, aunque sea metadato puro y tarde 3 ms, necesita un **bloqueo exclusivo de esquema**. Si en ese momento hay un informe de ventas leyendo `Pedidos` desde hace cuatro minutos, el `ALTER` no puede tomarlo y **espera**. Hasta aquí, nada grave. Lo grave es lo que pasa mientras espera: **todas las consultas nuevas se ponen detrás de él en la cola**, aunque solo quisieran leer una fila. El gestor de bloqueos no las adelanta, porque adelantarlas dejaría al `ALTER` esperando para siempre.

```
Informe de ventas (4 min)  ──────────────────────────────────►
ALTER TABLE (3 ms)              [esperando] ─────────────────► ✓
SELECT del catálogo                  [esperando] ────────────►
SELECT del catálogo                       [esperando] ───────►
SELECT del catálogo                            [esperando] ──►
```

Un `ALTER TABLE` de 3 ms ha dejado el servicio caído los cuatro minutos que le quedaban al informe. Y en los logs de la aplicación no hay errores de SQL: solo *timeouts* de peticiones HTTP, que es lo que hace tan difícil de diagnosticar el incidente. La protección es **no esperar indefinidamente**: se fija un tiempo máximo para adquirir el bloqueo y se reintenta.

```sql
SET LOCK_TIMEOUT 3000;   -- milisegundos; por defecto es -1, esperar siempre
ALTER TABLE Productos ADD NombreProducto NVARCHAR(200) NULL;
```

Si en tres segundos no consigue el bloqueo, el `ALTER` se rinde con este error y la cola se disuelve al instante:

```
Msg 1222, Level 16, State 56, Line 2
Lock request time out period exceeded.
```

Eso es exactamente lo que quieres: mejor que falle la migración que el servicio. Con el reintento alrededor:

```sql
DECLARE @intento INT = 0;
WHILE @intento < 5
BEGIN
    BEGIN TRY
        SET LOCK_TIMEOUT 3000;
        ALTER TABLE Productos ADD NombreProducto NVARCHAR(200) NULL;
        BREAK;
    END TRY
    BEGIN CATCH
        IF ERROR_NUMBER() <> 1222 THROW;   -- si no es timeout de bloqueo, propágalo
        SET @intento += 1;
        WAITFOR DELAY '00:00:10';
    END CATCH
END
```

En **PostgreSQL** es `SET lock_timeout = '3s'`, y el error es `ERROR: canceling statement due to lock timeout`. No lo confundas con `statement_timeout`, que limita la ejecución total y no la espera por el bloqueo: si pones solo el segundo, el `ALTER` puede consumirlo entero esperando y arrastrar la cola con él.

## Restricciones sin validar los datos existentes

Añadir una clave ajena o un `CHECK` tiene dos costes separados: registrar la restricción (instantáneo) y comprobar que las filas que ya existen la cumplen (un escaneo completo bajo bloqueo). Los motores permiten separarlos. En SQL Server, con `WITH NOCHECK`:

```sql
ALTER TABLE Pedidos WITH NOCHECK
ADD CONSTRAINT FK_Pedidos_Clientes FOREIGN KEY (ClienteId) REFERENCES Clientes(Id);
```

**Qué garantiza:** que a partir de ahora ningún `INSERT` ni `UPDATE` cree un pedido con un `ClienteId` inexistente. **Qué no garantiza:** que los pedidos que ya estaban no sean huérfanos. Y tiene un efecto secundario que pocos ven venir: la restricción queda marcada como *no confiable* y el optimizador **deja de usarla** para eliminar `JOIN` innecesarios, así que algunas consultas empeoran. Se detectan con `SELECT name, is_not_trusted FROM sys.foreign_keys WHERE is_not_trusted = 1;`, y para recuperar la confianza hay que validar —`ALTER TABLE Pedidos WITH CHECK CHECK CONSTRAINT FK_Pedidos_Clientes;`—, que sí escanea y va con `LOCK_TIMEOUT` puesto.

En **PostgreSQL** el patrón equivalente es mejor, porque la validación tampoco bloquea:

```sql
-- Instantáneo: registra la restricción, no mira las filas existentes
ALTER TABLE pedidos ADD CONSTRAINT fk_pedidos_clientes
    FOREIGN KEY (cliente_id) REFERENCES clientes(id) NOT VALID;

-- Escanea con SHARE UPDATE EXCLUSIVE: lecturas y escrituras siguen funcionando
ALTER TABLE pedidos VALIDATE CONSTRAINT fk_pedidos_clientes;
```

En los dos motores conviene saber antes a qué te enfrentas: si `SELECT COUNT(*) FROM Pedidos P LEFT JOIN Clientes C ON C.Id = P.ClienteId WHERE C.Id IS NULL` no devuelve `0`, la validación va a fallar y hay que decidir qué se hace con esos pedidos.

## Lo que le toca al código

La mitad del trabajo de una migración zero-downtime no está en el SQL. Son tres cosas:

- **Doble escritura.** Mientras conviven las dos columnas, cada escritura las actualiza a la vez, en un solo sitio del código y dentro de la misma transacción. Si están en dos métodos distintos, tarde o temprano alguien añade un camino que solo actualiza una.
- **Lectura tolerante.** El código nuevo tiene que aceptar que la columna nueva puede venir a `null` en las filas que el *backfill* aún no ha tocado. Eso es el `?? producto.Nombre` de antes, y no es un parche: es reconocer que durante la transición los datos están en dos estados.
- ***Feature flags*.** Un flag separa "el código está desplegado" de "el código usa lo nuevo". Sin flag, volver atrás exige un despliegue de emergencia; con flag, es apagar un interruptor en segundos sin tocar la base de datos. Además te deja hacer el corte cuando el sistema está tranquilo, no cuando termina el *pipeline*.

## El paso «contract» que nunca llega

Es lo más habitual del mundo: se hace el *expand*, se hace el *migrate*, el sistema funciona, y la columna `Nombre` sigue ahí tres años después. Nadie la borra porque nadie está seguro de que no la use algo. Lo que hace que se complete de verdad son dos cosas concretas. La primera es **una tarea creada en el mismo momento, con fecha**: no "cuando haya tiempo", sino una tarea abierta en el mismo *commit* que el *expand*, con fecha objetivo y con el `DROP COLUMN` ya escrito dentro. La migración no está terminada hasta que esa tarea se cierra.

La segunda es **una comprobación de que nadie la usa**. El código propio se busca con un `grep`, pero también hay que mirar vistas, procedimientos almacenados y las consultas que llegan desde fuera:

```sql
SELECT OBJECT_NAME(object_id) AS objeto FROM sys.sql_modules WHERE definition LIKE '%Nombre%';
SELECT DISTINCT query_sql_text FROM sys.query_store_query_text WHERE query_sql_text LIKE '%Nombre%';
```

Dos límites honestos: el Query Store solo guarda su periodo de retención (30 días por defecto), así que un informe trimestral se le escapa, y `LIKE '%Nombre%'` también encuentra `NombreProducto`, así que los resultados hay que leerlos, no contarlos.

## Cómo se verifica que la convivencia funciona

Esta es la comprobación que casi nadie hace y la única que responde a la pregunta real. No es "¿pasan las pruebas del código nuevo contra el esquema nuevo?" —eso solo valida el estado final—, sino:

> **Ejecuta la suite de pruebas de la versión anterior del código contra el esquema nuevo.** Si pasa, el paso es seguro.

Es exactamente la combinación que va a existir en producción durante la ventana de convivencia, y la que existirá durante horas si hay que hacer *rollback*. En la práctica: restaura una copia de `TiendaDB`, aplica la migración nueva sobre ella, y lanza las pruebas desde la etiqueta de git de la versión que hay ahora en producción.

```bash
# contra una copia de TiendaDB con la migración nueva ya aplicada
git checkout v1.14.0        # la versión que está en producción ahora mismo
dotnet test
```

Móntalo como un paso del *pipeline* en cada *pull request* que toque una migración: es la diferencia entre creer que un cambio es compatible hacia atrás y saberlo.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| Errores 500 en parte de las peticiones durante los minutos del despliegue, con `Invalid column name 'Nombre'.` en el log | El `DROP COLUMN` se hizo antes de que todas las instancias dejaran de leerla: el *contract* adelantado |
| `Cannot insert the value NULL into column 'NumeroSeguimiento', table 'TiendaDB.dbo.Pedidos'; column does not allow nulls. INSERT fails.` | Se endureció a `NOT NULL` cuando todavía había instancias v1 insertando sin rellenarla |
| El despliegue se queda colgado y **todo** el servicio deja de responder; en la aplicación solo hay *timeouts*, ningún error de SQL | Cola de bloqueos: el `ALTER TABLE` espera detrás de una consulta larga y encola a todo lo que llega después |
| `The transaction log for database 'TiendaDB' is full due to 'ACTIVE_TRANSACTION'.` | *Backfill* en una sola transacción: nada del log se recicla hasta el `COMMIT` |
| `Invalid column name 'Nombre'.` justo después de un `sp_rename` que "se ejecutó sin errores" | Un renombrado tratado como atómico. Lo es para el motor, no para el código que ya está desplegado |
| `String or binary data would be truncated in table 'TiendaDB.dbo.Productos', column 'NombreProducto'.` | Se redujo la longitud sin comprobar antes cuántas filas exceden la nueva |
| `The ALTER TABLE statement conflicted with the FOREIGN KEY constraint "FK_Pedidos_Clientes".` | Hay pedidos huérfanos: la clave ajena valida los datos existentes salvo que uses `WITH NOCHECK` / `NOT VALID` |
| PostgreSQL: el índice aparece en `\d pedidos` pero `EXPLAIN` no lo usa y las escrituras van más lentas | Un `CREATE INDEX CONCURRENTLY` falló y dejó el índice `INVALID`; hay que borrarlo antes de reintentar |
| PostgreSQL: `ERROR: canceling statement due to lock timeout` | La protección funcionando. No es un fallo que arreglar, es un reintento que programar |

## Cuándo NO merece la pena

Todo esto tiene un precio, y no es pequeño: un renombrado de columna pasa de ser un `sp_rename` de una línea a ser tres despliegues, un *backfill* por lotes, doble escritura en el código, una lectura con *fallback* y una tarea de limpieza pendiente. Semanas de calendario para algo que en cinco minutos de parada es un script.

Si tienes una ventana de mantenimiento aceptable —una tienda cuyo tráfico entre las 4 y las 5 de la mañana es de tres visitas, un sistema interno que nadie usa los domingos—, **úsala**. Parar, aplicar el cambio directo, arrancar y verificar es más simple, más rápido y tiene muchas menos formas de salir mal que una convivencia de tres pasos mal terminada. Que se pueda hacer sin parar no significa que haya que hacerlo sin parar. Y aunque no tengas ventana, no todo cambio necesita el mismo rigor: añadir una columna *nullable* es directo siempre, y el aparato completo de *expand/migrate/contract* se reserva para la mitad peligrosa de la tabla de clasificación.

## Buenas prácticas avanzadas

- **Aplica siempre "*nullable* o con `DEFAULT`" antes que "`NOT NULL`", y en PostgreSQL usa el truco del `CHECK NOT VALID`.** Añadir una columna obligatoria de golpe es la forma más común de romper un despliegue, porque falla justo en el instante en que conviven versiones. Y el endurecimiento posterior tampoco es gratis: en SQL Server valida o reescribe la tabla bajo bloqueo, mientras que en PostgreSQL un `CHECK` ya validado hace que el `SET NOT NULL` sea metadato.
- **Nunca borres ni renombres en el mismo paso que el cambio de código.** Un `sp_rename` o un `RenameColumn` de EF Core parece atómico, y lo es para el motor, pero si el código viejo sigue esperando el nombre anterior revienta igual que si hubieras borrado la columna. Trata cualquier renombrado como un *expand/migrate/contract* completo.
- **En producción, todo DDL con `LOCK_TIMEOUT` y reintento.** Es la única protección contra el fallo que más daño hace: un `ALTER TABLE` de milisegundos que espera detrás de una consulta larga y encola al servicio entero. Que la migración falle y se reintente es infinitamente mejor que una caída de cuatro minutos que en los logs solo aparece como *timeouts*.
- **Recuerda que el *rollback* del código también tiene que ser compatible.** La base de datos avanza hacia delante y el código puede volver atrás: esa asimetría obliga a que el esquema tolere la versión anterior **indefinidamente**, no solo durante los tres minutos del despliegue. Si un cambio solo es compatible mientras el despliegue avanza, no es compatible. Los *feature flags* son la herramienta que hace ese *rollback* barato: reviertes un flag en segundos en lugar de una migración en frío, y el corte lo haces cuando tú decides.
- **Trata el *contract* como parte de la migración, con fecha y con la consulta que lo justifica.** Es habitual completar *expand* y *migrate*, ver el sistema funcionando y no volver nunca a limpiar. Una tarea abierta en el mismo *commit*, con el `DROP COLUMN` ya escrito y una comprobación de uso real, es lo único que hace que ocurra.

## Recursos didácticos

- [Parallel Change, de Martin Fowler](https://martinfowler.com/bliki/ParallelChange.html) — la descripción original y general del patrón *expand/migrate/contract*, más allá de las bases de datos. Corto y clarificador.
- [Strong Migrations](https://github.com/ankane/strong_migrations) — una librería de Ruby que rechaza en CI las migraciones peligrosas. Su README es la mejor lista pública de "este cambio bloquea, y esta es la alternativa segura", con el detalle por versión de motor, y se lee perfectamente aunque no toques Ruby.
- [Explicit Locking, en la documentación de PostgreSQL](https://www.postgresql.org/docs/current/explicit-locking.html) — la matriz de qué bloqueos son incompatibles entre sí. Es lo que explica, de forma exacta y nada intuitiva, por qué se forma la cola.
- [Online index operations, en la documentación de SQL Server](https://learn.microsoft.com/sql/relational-databases/indexes/perform-index-operations-online) — qué operaciones admiten `ONLINE = ON`, con qué ediciones y qué bloqueos toma cada una en cada fase.

---

*En resumen: zero-downtime no es una herramienta sino una disciplina — dividir cada cambio de esquema en pasos que nunca dejan de ser compatibles con el código que convive con ellos, incluido el código al que podrías tener que volver.*
