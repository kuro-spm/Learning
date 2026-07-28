# Microsoft.Data.SqlClient

## ¿Qué es?

Es el driver oficial de Microsoft para conectar aplicaciones .NET con SQL Server: abre la conexión TCP, habla el protocolo TDS, envía sentencias y devuelve filas. Es la implementación de ADO.NET (`SqlConnection`, `SqlCommand`, `SqlDataReader`) sobre la que se apoyan Entity Framework Core y Dapper.

## ¿Por qué existe?

`System.Data.SqlClient` venía dentro del framework, y eso era exactamente el problema: su ciclo de vida estaba atado al de .NET. Cualquier funcionalidad nueva —Always Encrypted, autenticación con Microsoft Entra ID (el antiguo Azure AD), TDS 8— tenía que esperar a una versión del framework y, además, mantener compatibilidad binaria con código de 2005. Microsoft lo declaró **congelado**: recibe parches de seguridad y nada más.

`Microsoft.Data.SqlClient` es el mismo driver sacado a un paquete NuGet independiente, con su propio versionado. Puede romper valores por defecto entre versiones mayores (y lo ha hecho), publicar cada pocos meses y soportar características que el driver antiguo nunca tendrá.

> Si vienes de otro ecosistema, piensa en esto como la separación entre el JDK y el `mysql-connector-java`: el driver deja de ser parte de la plataforma y pasa a ser una dependencia que tú actualizas.

La conclusión práctica es corta: **si estás en .NET 6+, usas `Microsoft.Data.SqlClient`, punto.** El otro solo aparece en código que nadie ha tocado desde .NET Framework.

## ¿Cuándo y para qué se usa?

En cualquier backend .NET que hable con SQL Server. El ejemplo conductor de esta guía es una **tienda online** con base de datos `TiendaDB`, tablas `Productos`, `Pedidos` y `Clientes`, y un pedido concreto que iremos siguiendo: el **#4711**.

Casi nunca lo usarás en crudo para el CRUD del día a día —para eso están [Dapper](Dapper.md) y [Entity Framework Core](Entity-Framework-Core.md), que van por encima de este mismo driver—, pero sí para lo que ellos no cubren bien: cargas masivas con `SqlBulkCopy`, procedimientos con varios *result sets*, control fino de tipos y tamaños de parámetro, o depurar por qué una consulta que en SSMS tarda 3 ms desde la aplicación tarda 800.

Y aunque nunca escribas un `SqlCommand` a mano, la cadena de conexión, el *pooling* y los errores transitorios son suyos. Todo lo que falla en la capa de datos de un backend .NET falla aquí.

---

## Instalación y el paquete que hay que dejar atrás

```bash
dotnet add package Microsoft.Data.SqlClient
```

Las diferencias que importan frente al paquete antiguo no son de API —los tipos se llaman igual y tienen los mismos miembros— sino de comportamiento y de mantenimiento:

| | `System.Data.SqlClient` | `Microsoft.Data.SqlClient` |
|---|---|---|
| Mantenimiento | Congelado, solo seguridad | Activo, versiones cada pocos meses |
| `Encrypt` por defecto | `False` | `True` desde la versión 4.0 |
| Always Encrypted | Básico, sin *enclaves* | Completo, con *secure enclaves* |
| Autenticación Entra ID | ❌ No | ✅ `Authentication=Active Directory Default`, *managed identity*, MFA |
| TDS 8 (`Encrypt=Strict`) | ❌ No | ✅ Desde la 5.0 |
| `Command Timeout` en la cadena | ❌ No existe la palabra clave | ✅ Sí |
| Reintentos configurables en el driver | ❌ No | ✅ `SqlRetryLogicOption` desde la 3.0 |
| Tipos nuevos del motor (`UTF8`, JSON) | Se quedan atrás | Se van añadiendo |

**Qué hay que cambiar al migrar, más allá del `using`.** Sustituir `using System.Data.SqlClient;` por `using Microsoft.Data.SqlClient;` es el 90 % de las ediciones y el 10 % del trabajo. Lo que de verdad hay que revisar:

- **El valor por defecto de `Encrypt`.** Es el que rompe la migración, y tiene su propia sección más abajo.
- **`using System.Data;` se queda.** `SqlDbType`, `DataTable`, `IsolationLevel` y `System.Data.SqlTypes` siguen donde estaban: los comparten los dos drivers. Solo se mueven los tipos con prefijo `Sql` del proveedor (`SqlConnection`, `SqlCommand`, `SqlParameter`, `SqlException`, `SqlBulkCopy`).
- **Las dependencias transitivas.** Si una librería de terceros expone en su API un `System.Data.SqlClient.SqlConnection`, no puedes pasarle el tuyo: son tipos distintos con el mismo nombre. Toca actualizar esa librería o envolverla. Y si resuelves el proveedor por nombre en tiempo de ejecución, el nombre invariante también cambia: `DbProviderFactories.RegisterFactory("Microsoft.Data.SqlClient", SqlClientFactory.Instance)`.
- **`Application Name` por defecto.** Pasa de `.NET SqlClient Data Provider` a `Core Microsoft SqlClient Data Provider`. Si tienes alertas o filtros de auditoría que buscan ese literal, se quedan ciegos sin que nada falle.

## La cadena de conexión, parámetro a parámetro

La cadena de conexión es el fichero de configuración más importante del driver y el que más gente copia de un ejemplo de Stack Overflow sin leer. Esta es la de desarrollo de la tienda:

```json
"ConnectionStrings": {
  "TiendaDB": "Server=localhost,1433;Database=TiendaDB;User ID=tienda_app;Password=***;Encrypt=True;TrustServerCertificate=True;Connect Timeout=5;Command Timeout=30;Application Name=Tienda.Api;Max Pool Size=50;MultipleActiveResultSets=False"
}
```

| Parámetro | Qué hace | Qué valor poner | Qué se rompe si lo dejas mal |
|---|---|---|---|
| `Server` | Host, instancia y puerto | `host,1433` — la coma es el puerto; la barra invertida es instancia nombrada | Con instancia nombrada y sin puerto necesitas el SQL Browser (UDP 1434) abierto |
| `Database` | Base de datos inicial | `TiendaDB` siempre explícita | Sin ella entras en la base por defecto del usuario: `Invalid object name 'Productos'` |
| `User ID` / `Password` | Autenticación SQL | Solo si no hay Windows ni Entra ID | Contraseña en texto plano si no usas un gestor de secretos |
| `Trusted_Connection=True` | Autenticación integrada de Windows | En dominio, es lo preferible: no hay contraseña que rotar | En Linux o contenedor no hay identidad de Windows: falla el login |
| `Integrated Security=True` | Sinónimo del anterior | Cualquiera de los dos, no ambos | Nada, son equivalentes |
| `Encrypt` | Cifra el canal TLS | `True` (por defecto desde 4.0). `Strict` si el servidor soporta TDS 8 | Ver la sección siguiente: es el fallo número uno al migrar |
| `TrustServerCertificate` | Acepta el certificado del servidor sin validarlo | `True` **solo** en desarrollo local | En producción abre la puerta a *man-in-the-middle* |
| `Connect Timeout` | Segundos esperando a **abrir** la conexión | 5-15. Por defecto 15 | Alto: una base caída deja peticiones colgadas 15 s cada una y agota el pool |
| `Command Timeout` | Segundos esperando a que **termine** una sentencia | 30 (el defecto). Súbelo por comando, no aquí | Bajo: consultas legítimas mueren con error `-2`. Alto: una consulta atascada bloquea el hilo minutos |
| `Application Name` | Cadena que ve el servidor en `sys.dm_exec_sessions` | El nombre real del servicio: `Tienda.Api` | Sin él, todas tus aplicaciones son la misma en el monitor: imposible saber quién lanzó la consulta lenta |
| `Pooling` | Reutiliza conexiones físicas | `True` siempre, salvo en un script de un solo uso | `False` cuesta unos 30-50 ms de *handshake* TLS y login en **cada** operación |
| `Min Pool Size` | Conexiones que se mantienen abiertas | 0 en general; 2-5 si el arranque en frío importa | Con 0, la primera petición tras un rato de calma paga el coste completo de apertura |
| `Max Pool Size` | Techo de conexiones por pool | 100 por defecto. Bájalo si hay muchas instancias | Ver el cálculo de la sección de *pooling* |
| `MultipleActiveResultSets` | Permite varios lectores abiertos sobre una conexión | `False`. Actívalo solo si lo necesitas de verdad | Con `False`, dos lectores simultáneos dan `The connection does not support MultipleActiveResultSets` |

Una nota sobre `MultipleActiveResultSets` (MARS): parece la solución cómoda a "no puedo iterar un `SqlDataReader` y consultar dentro del bucle", pero eso es casi siempre un problema de diseño —el clásico N+1— y activar MARS lo esconde en lugar de arreglarlo. Además MARS tiene interacciones molestas con transacciones y desactiva algunas optimizaciones del driver. Léelo como un interruptor de compatibilidad, no como una mejora.

## `Encrypt=True` por defecto: el cambio que rompe migraciones

Desde `Microsoft.Data.SqlClient` **4.0**, `Encrypt` vale `True` si no dices lo contrario. En el driver antiguo valía `False`. Es un cambio deliberado —cifrar por defecto es lo correcto— y es la causa de que una aplicación que funcionaba deje de arrancar en cuanto cambias el paquete, sin haber tocado la cadena de conexión:

```
A connection was successfully established with the server, but then an error occurred
during the login process. (provider: SSL Provider, error: 0 - The certificate chain was
issued by an authority that is not trusted.)
```

Fíjate en lo que dice: la conexión **sí** se estableció. El fallo es posterior, en el *handshake* TLS: el cliente pide cifrado, el servidor presenta su certificado y el cliente no reconoce a quien lo firmó. Un SQL Server recién instalado usa un certificado autofirmado, así que esto pasa en el 100 % de los entornos locales.

Hay dos salidas, y no son equivalentes:

```csharp
// ❌ El parche: dejar de validar el certificado
"Server=sql-prod;Database=TiendaDB;Encrypt=True;TrustServerCertificate=True"

// ✅ La solución: que el certificado sea de confianza
"Server=sql-prod.tienda.local;Database=TiendaDB;Encrypt=True"
```

`TrustServerCertificate=True` **sigue cifrando el tráfico**, pero deja de comprobar con quién habla. Eso convierte el cifrado en decorativo: quien se ponga en medio de la red presenta su propio certificado, el cliente lo acepta sin protestar, y descifra y reenvía todo el tráfico —incluida la contraseña del login— sin que nadie note nada. Es un agujero de *man-in-the-middle* de manual. En desarrollo local, con cliente y servidor en la misma máquina, es aceptable; en producción es una brecha.

La vía correcta es instalar en el servidor un certificado emitido por una CA en la que el cliente ya confíe (la interna de la organización, o una pública) y usar en `Server` el **mismo nombre** que aparece en el certificado: si el certificado dice `sql-prod.tienda.local` y tú conectas a `10.0.4.12`, la validación falla igual. Desde la versión 5.0 hay además un tercer valor, `Encrypt=Strict`, que activa TDS 8 —el cifrado se negocia antes del login en lugar de a mitad— e ignora `TrustServerCertificate` por completo. Si el servidor lo soporta, es el objetivo al que migrar.

## Connection pooling: por qué abrir y cerrar es lo correcto

Abrir una conexión a SQL Server cuesta un *handshake* TCP, otro TLS y un login: entre 30 y 50 ms en red local. Si eso pasara en cada consulta, ningún backend aguantaría. No pasa porque el driver mantiene un **pool**: cuando haces `Dispose()` de una `SqlConnection`, la conexión física no se cierra, vuelve al pool limpia y lista para el siguiente que la pida.

Esto invierte la intuición de mucha gente. El patrón correcto **no** es guardar una `SqlConnection` abierta en un campo del repositorio y reutilizarla: esa conexión no es *thread-safe*, se queda inservible tras un error de red sin que nada la reponga, arrastra bloqueos y contexto de sesión entre operaciones que no tienen nada que ver, y —lo más importante— impide que el pool haga su trabajo. Lo correcto es lo que parece derrochador:

```csharp
// ✅ Abrir y cerrar en cada operación: el pool lo hace gratis
public async Task<decimal> ObtenerTotalAsync(int pedidoId, CancellationToken ct)
{
    await using var conn = new SqlConnection(_cadena);
    await conn.OpenAsync(ct);

    await using var cmd = new SqlCommand("SELECT Total FROM Pedidos WHERE Id = @id", conn);
    cmd.Parameters.Add("@id", SqlDbType.Int).Value = pedidoId;

    return (decimal)await cmd.ExecuteScalarAsync(ct);
}
```

`OpenAsync` aquí no abre nada la mayoría de las veces: coge una conexión ya viva del pool. El `await using` la devuelve. Cada operación es corta, aislada y segura entre hilos.

**Los pools se agrupan por cadena de conexión exacta.** No por servidor ni por base de datos: por la cadena, comparada carácter a carácter tras normalizar las palabras clave, más la identidad usada. Dos cadenas que difieran en un solo carácter son dos pools independientes, cada uno con su propio `Max Pool Size`.

De ahí sale un error caro y silencioso:

```csharp
// ❌ Un pool nuevo por cada valor de tenant
var cadena = $"Server=sql-prod;Database=TiendaDB;User ID={usuario};Password={clave};Application Name=Tienda-{tenantId}";
```

Con 100 valores distintos de `tenantId`, tienes 100 pools de hasta 100 conexiones: 10 000 conexiones potenciales contra un servidor dimensionado para unos cientos. Y el ahorro del pool desaparece, porque cada pool nuevo arranca vacío. La cadena de conexión se construye **una vez** y se inyecta; lo que varía por petición va en parámetros del comando, nunca en la cadena.

**Cuando el pool se agota**, el driver no falla al instante: encola la petición y espera hasta `Connect Timeout`. Si en ese tiempo nadie devuelve una conexión, salta esto:

```
Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool.
This may have occurred because all pooled connections were in use and max pool size was reached.
```

Y aquí está el cálculo que hace tangible el techo. Con `Max Pool Size=100` (el valor por defecto) y consultas que tardan 200 ms:

```
Cada conexión atiende:   1 s / 0,2 s        =   5 operaciones por segundo
Con 100 conexiones:      100 × 5            = 500 operaciones por segundo
```

**500 peticiones por segundo** es el techo antes de que empiece la cola. A 600 req/s la cola crece indefinidamente y en pocos segundos los primeros en esperar revientan con el error de arriba. Fíjate en la palanca real: bajar la consulta de 200 ms a 20 ms multiplica el techo por diez sin tocar el pool. Subir `Max Pool Size` a 500 no lo hace, porque el cuello se traslada al servidor —que ahora tiene 500 consultas compitiendo por la misma CPU y los mismos bloqueos— y las consultas se vuelven más lentas todavía. Es decir: un `Max Pool Size` grande **oculta** un problema de consultas lentas y lo convierte en un problema de servidor saturado. Y con varias instancias detrás de un balanceador el límite real es `Max Pool Size × número de instancias`, así que el defecto de 100 desborda al servidor mucho antes de lo que parece.

## `SqlCommand` y parámetros: por qué `AddWithValue` es una trampa

Parametrizar no es opcional: es lo que separa una consulta de una inyección de SQL. Pero *cómo* se parametriza tiene consecuencias de rendimiento que no aparecen en ningún tutorial.

`AddWithValue` es la forma cómoda, y hace algo que no le has pedido: **inferir el tipo** a partir del valor de C#.

```csharp
// ❌ El tipo lo decide el driver
cmd.Parameters.AddWithValue("@sku", "ZAP-42");
```

Un `string` de C# se infiere como `nvarchar`, y el tamaño se toma de la longitud del valor. Si la columna `Productos.Sku` está declarada como `varchar(50)`, acabas de crear una comparación entre `varchar` y `nvarchar`. Y en SQL Server `nvarchar` tiene **precedencia de tipo más alta** que `varchar`, así que la conversión no se aplica al parámetro: se aplica a la columna, fila por fila.

Con la columna convertida, el índice sobre `Sku` deja de ser aplicable. El plan pasa de un *seek* a un *scan*:

```
-- ❌ Con AddWithValue
Index Scan (Productos.IX_Productos_Sku)
  Predicate: CONVERT_IMPLICIT(nvarchar(50),[Productos].[Sku],0) = [@sku]

-- ✅ Con el tipo declarado
Index Seek (Productos.IX_Productos_Sku)
  Seek Keys: Prefix: [Productos].[Sku] = 'ZAP-42'
```

`CONVERT_IMPLICIT` sobre una columna dentro de un predicado es la señal literal de este problema, y en una tabla de un millón de productos es la diferencia entre leer una página y leer la tabla entera. La forma correcta declara tipo y tamaño:

```csharp
// ✅ Tipo y tamaño explícitos: el parámetro coincide con la columna
cmd.Parameters.Add("@sku", SqlDbType.VarChar, 50).Value = "ZAP-42";
cmd.Parameters.Add("@total", SqlDbType.Decimal, 0).Value = 79.90m;
cmd.Parameters["@total"].Precision = 10;
cmd.Parameters["@total"].Scale = 2;
```

El tamaño explícito arregla además un segundo problema más sutil. `AddWithValue` dimensiona el parámetro según el valor concreto, así que `"ZAP-42"` genera un `nvarchar(6)` y `"ZAPATILLA-RUNNING-42"` un `nvarchar(20)`. Para SQL Server son **dos consultas distintas**, cada una con su plan en la caché: un endpoint de búsqueda con SKUs de longitudes variadas genera decenas de planes para la misma sentencia y fuerza compilaciones que deberían reutilizarse. Con `SqlDbType.VarChar, 50` fijo, el plan es siempre el mismo. Y con `decimal` y `DateTime` el riesgo es peor que el rendimiento: la precisión y la escala también se deducen del valor, y un total como `79.905m` puede llegar al servidor redondeado sin que nada avise.

Regla práctica: **`AddWithValue` solo para `int`, `bool` y `Guid`**, donde no hay tamaño que inferir. Para cadenas, decimales y fechas, `Add` con tipo. Los nulos tienen su propia trampa, porque `null` de C# no es `NULL` de SQL:

```csharp
// ❌ AddWithValue con null lanza: "Parameterized Query ... expects the parameter '@nota',
//    which was not supplied."  — el parámetro se envía sin valor
cmd.Parameters.AddWithValue("@nota", pedido.Nota);

// ✅
cmd.Parameters.Add("@nota", SqlDbType.NVarChar, 200).Value = (object?)pedido.Nota ?? DBNull.Value;
```

## Leer resultados con `SqlDataReader`

`ExecuteReaderAsync` devuelve un cursor de solo avance sobre las filas: no carga el resultado en memoria, va pidiendo al servidor a medida que lo recorres.

```csharp
await using var cmd = new SqlCommand(
    "SELECT Id, Nombre, Precio, Descripcion FROM Productos WHERE Activo = 1", conn);
await using var reader = await cmd.ExecuteReaderAsync(ct);

int iId = reader.GetOrdinal("Id"), iNombre = reader.GetOrdinal("Nombre");
int iPrecio = reader.GetOrdinal("Precio"), iDescripcion = reader.GetOrdinal("Descripcion");

var productos = new List<Producto>();
while (await reader.ReadAsync(ct))
{
    productos.Add(new Producto
    {
        Id = reader.GetInt32(iId),
        Nombre = reader.GetString(iNombre),
        Precio = reader.GetDecimal(iPrecio),
        Descripcion = await reader.IsDBNullAsync(iDescripcion, ct) ? null
                    : await reader.GetFieldValueAsync<string>(iDescripcion, ct)
    });
}
```

Tres decisiones en ese fragmento:

- **`GetOrdinal` fuera del bucle.** `reader["Nombre"]` funciona, pero resuelve el nombre a posición en cada fila y devuelve `object`, así que además boxea los valores. Con 50 000 filas y cuatro columnas eso son 200 000 búsquedas y 200 000 *boxings* evitables. Resolver los ordinales una vez y usar `GetInt32(i)` es el patrón de los micro-ORM por algo.
- **`IsDBNull` antes de leer.** Un `GetString` sobre una columna nula lanza `SqlNullValueException`. No hay atajo: o compruebas, o usas `GetFieldValue<T>` con un tipo que admita nulos.
- **`GetFieldValueAsync<T>` frente a `GetFieldValue<T>`.** Solo aporta si la columna es grande (`nvarchar(max)`, `varbinary(max)`) y abriste el lector con `CommandBehavior.SequentialAccess`: entonces el valor puede llegar en varios paquetes de red y la versión asíncrona no bloquea el hilo esperándolos. Para un `int` o un `varchar(50)` el dato ya está en el buffer y la versión síncrona es más rápida.

**Varios result sets.** Un procedimiento almacenado puede devolver el pedido y sus líneas en una sola ida y vuelta. `NextResultAsync` avanza al siguiente:

```csharp
await using var cmd = new SqlCommand("dbo.ObtenerPedidoConLineas", conn)
    { CommandType = CommandType.StoredProcedure };
cmd.Parameters.Add("@pedidoId", SqlDbType.Int).Value = 4711;
await using var reader = await cmd.ExecuteReaderAsync(ct);

// Primer result set: la cabecera del pedido #4711
await reader.ReadAsync(ct);
var pedido = new Pedido { Id = reader.GetInt32(0), Total = reader.GetDecimal(1) };

// Segundo result set: sus líneas
await reader.NextResultAsync(ct);
while (await reader.ReadAsync(ct))
    pedido.Lineas.Add(new LineaPedido { ProductoId = reader.GetInt32(0), Unidades = reader.GetInt32(1) });
```

Esta es la alternativa correcta a MARS: un viaje, un lector, dos resultados en orden.

## Asincronía de verdad

Todas las operaciones tienen su par asíncrono: `OpenAsync`, `ExecuteReaderAsync`, `ExecuteNonQueryAsync`, `ExecuteScalarAsync`, `ReadAsync`, `CommitAsync`. Y todas aceptan un `CancellationToken` que hay que pasar, no ignorar: en ASP.NET Core, el token de la petición se cancela cuando el cliente cierra la conexión, y propagarlo hasta aquí hace que el driver envíe un `ATTENTION` al servidor para abortar la consulta en curso. Sin él, el usuario se ha ido y el servidor sigue trabajando para nadie. Lo que no se puede hacer es mezclar:

```csharp
// ❌ Bloquea un hilo del pool durante toda la consulta
var total = cmd.ExecuteScalarAsync().Result;
conn.OpenAsync().Wait();
```

Una consulta de 200 ms con `.Result` es un hilo del *thread pool* de .NET parado 200 ms sin hacer nada. El pool de hilos crece a razón de aproximadamente **un hilo nuevo por segundo** cuando detecta escasez, así que un pico de tráfico produce el cuadro clásico: la CPU al 10 %, la base de datos ociosa y los tiempos de respuesta en varios segundos. Es *thread pool starvation*, y en un backend con acceso a datos síncrono es la causa habitual. Además, con `.Result` la excepción llega envuelta en `AggregateException` y los `catch (SqlException)` de arriba dejan de capturar nada.

## Transacciones

Se abre sobre la conexión y **hay que asignarla al comando**:

```csharp
await using var conn = new SqlConnection(_cadena);
await conn.OpenAsync(ct);
await using var tx = (SqlTransaction)await conn.BeginTransactionAsync(IsolationLevel.ReadCommitted, ct);

try
{
    // el tercer argumento del constructor es la transacción
    await using var cmdStock = new SqlCommand(
        "UPDATE Productos SET Stock = Stock - @u WHERE Id = @id AND Stock >= @u", conn, tx);
    cmdStock.Parameters.Add("@u", SqlDbType.Int).Value = 2;
    cmdStock.Parameters.Add("@id", SqlDbType.Int).Value = 91;
    if (await cmdStock.ExecuteNonQueryAsync(ct) == 0)
        throw new InvalidOperationException("Sin stock suficiente");

    await using var cmdPedido = new SqlCommand(
        "UPDATE Pedidos SET Estado = 'confirmado' WHERE Id = 4711", conn, tx);
    await cmdPedido.ExecuteNonQueryAsync(ct);

    await tx.CommitAsync(ct);
}
catch
{
    await tx.RollbackAsync(ct);
    throw;
}
```

Si olvidas el tercer argumento del constructor (o `cmd.Transaction = tx`), no se ejecuta fuera de la transacción: falla en seco.

```
InvalidOperationException: ExecuteNonQuery requires the command to have a transaction
when the connection assigned to the command is in a pending local transaction.
The Transaction property of the command has not been initialized.
```

Es un buen error, porque hace imposible el fallo silencioso, y explica por qué los micro-ORM piden la transacción como parámetro en cada llamada.

Sobre los niveles: SQL Server usa `READ COMMITTED` con bloqueos por defecto, donde una lectura espera a la escritura que tenga la fila. Los útiles en la práctica son:

| Nivel | Qué garantiza | Cuándo |
|---|---|---|
| `ReadCommitted` | No lees cambios sin confirmar | El defecto, casi siempre |
| `Snapshot` | Instantánea consistente sin bloquear a nadie | Informes largos sobre tablas muy escritas (requiere `ALLOW_SNAPSHOT_ISOLATION`) |
| `Serializable` | Ni fantasmas ni relecturas distintas | Cálculos que deben cuadrar; a cambio, muchos más *deadlocks* que reintentar |
| `ReadUncommitted` | Nada | Casi nunca: es el `WITH (NOLOCK)` disfrazado, y puede devolver filas duplicadas o a medio escribir |

Y la regla que no está en la API: **nunca dejes una llamada de red dentro de una transacción abierta**. Cobrar con una pasarela de pago entre el `BEGIN` y el `COMMIT` mantiene los bloqueos sobre `Productos` durante los segundos que tarde el tercero, y una conexión del pool ocupada esperando. Transacción corta para reservar, llamada fuera, transacción corta para confirmar.

## `SqlBulkCopy`: cuando N `INSERT` no valen

Cargar el catálogo de un proveedor son 50 000 filas en `Productos`. Con `INSERT` de fila en fila, el coste dominante no es el trabajo del servidor: son 50 000 idas y vueltas por la red, cada una con su latencia, su registro en el log de transacciones y su plan.

`SqlBulkCopy` usa el mecanismo de carga masiva de TDS: manda las filas en bloques y salta buena parte de ese trabajo por fila.

```csharp
using var bulk = new SqlBulkCopy(_cadena, SqlBulkCopyOptions.TableLock)
{
    DestinationTableName = "dbo.Productos",
    BatchSize = 5_000,
    BulkCopyTimeout = 120
};

// Mapear por nombre siempre: sin ColumnMappings se asume el orden de las columnas
foreach (var col in new[] { "Sku", "Nombre", "Precio" })
    bulk.ColumnMappings.Add(col, col);

await bulk.WriteToServerAsync(lectorDeCsv, ct);   // acepta IDataReader, DataTable o DataRow[]
```

El orden de magnitud típico en red local, para esas 50 000 filas:

```
50 000 INSERT individuales   ≈ 50 000 × ~1,5 ms  ≈ 75 s
SqlBulkCopy, BatchSize=5000  ≈ 1-2 s
```

Unas **40-70 veces** más rápido, y la diferencia crece con la latencia de red. El umbral a partir del cual compensa está en unos pocos miles de filas; por debajo de mil, un `INSERT` con varios `VALUES` o un parámetro de tipo tabla es más simple y suficiente.

Dos avisos. `SqlBulkCopyOptions.TableLock` acelera mucho pero bloquea la tabla entera: úsalo en cargas nocturnas, no mientras la tienda vende. Y por defecto **no dispara los triggers ni comprueba las restricciones `CHECK`** —hay que pedirlo con `FireTriggers` y `CheckConstraints`—, así que puedes meter datos que la aplicación considera imposibles.

## Errores transitorios y resiliencia

Un **error transitorio** es un fallo que desaparece si vuelves a intentarlo: la base de datos estaba haciendo *failover*, el servicio te estaba limitando el caudal, la red parpadeó. No es un error de tu código, y tratarlo como un error de tu código —devolver un 500 al usuario— es tirar una petición perfectamente válida.

Los números que hay que conocer:

| Número | Significado | Reintentable |
|---|---|---|
| `-2` | Timeout de comando (`Command Timeout` agotado) | Sí, con cuidado: puede que la escritura sí se aplicara |
| `40613` | *Database is not currently available* (Azure SQL arrancando o migrando) | Sí |
| `49918` / `49919` | Throttling: no hay recursos para procesar la petición | Sí |
| `40197` / `40501` | Error del servicio durante una actualización, o servicio ocupado | Sí |
| `1205` | Deadlock: tu transacción fue la víctima elegida | Sí, reintenta la transacción **entera** |
| `4060` | *Cannot open database requested by the login* | No: base mal escrita o sin permisos |
| `18456` | *Login failed* | No: credenciales |
| `2627` / `2601` | Violación de clave única | No: es un dato duplicado |

El driver expone esto sin tablas mágicas: `SqlException.IsTransient` ya clasifica el error por ti, y `Number` te da el detalle.

```csharp
for (var intento = 1; ; intento++)
{
    try { return await ObtenerTotalAsync(4711, ct); }
    catch (SqlException ex) when (ex.IsTransient && intento < 4)
    {
        // 200 ms, 400 ms, 800 ms + jitter para no sincronizar todos los clientes
        var espera = TimeSpan.FromMilliseconds(200 * Math.Pow(2, intento - 1)
                                              + Random.Shared.Next(0, 100));
        _logger.LogWarning(ex, "Error transitorio {N}, reintento {I} en {Ms} ms",
                           ex.Number, intento, espera.TotalMilliseconds);
        await Task.Delay(espera, ct);
    }
}
```

El **backoff exponencial** es lo que distingue un reintento útil de un ataque a tu propia base de datos: si mil clientes reintentan a los 100 ms exactos, el servidor que se estaba recuperando recibe mil peticiones simultáneas y vuelve a caer. El *jitter* aleatorio rompe esa sincronización.

En código real no escribas ese bucle a mano. Hay tres opciones:

- **`Microsoft.Data.SqlClient` trae reintentos configurables** desde la 3.0: se define un `SqlRetryLogicOption` (número de intentos, intervalo, números de error) y se asigna a `SqlConnection.RetryLogicProvider` o `SqlCommand.RetryLogicProvider`. Actúa dentro del driver, sin envolver nada.
- **Polly** es la librería estándar de resiliencia en .NET, y se integra con `IHttpClientFactory` y con la tubería de ASP.NET Core. Su `ResiliencePipeline` cubre reintentos, *circuit breaker* y *timeout* con una sola política reutilizable.
- **Con EF Core no hace falta ninguna de las dos**: `EnableRetryOnFailure()` en la configuración activa la estrategia de ejecución del proveedor de SQL Server, que ya sabe qué números son transitorios.

```csharp
builder.Services.AddDbContext<TiendaDbContext>(options =>
    options.UseSqlServer(cadena, sql => sql.EnableRetryOnFailure(
        maxRetryCount: 3, maxRetryDelay: TimeSpan.FromSeconds(5), errorNumbersToAdd: null)));
```

Un detalle que sorprende a quien lo activa: con `EnableRetryOnFailure` no puedes llamar a `BeginTransaction` sin más, porque la estrategia necesita poder reejecutar el bloque completo. EF Core lo dice con un error explícito y obliga a envolverlo en `CreateExecutionStrategy().ExecuteAsync(...)`.

Y una advertencia que aplica a cualquier reintento: reintentar una **lectura** es gratis; reintentar una **escritura** no es idempotente. Si el timeout llegó después de que el servidor confirmara el `INSERT`, el reintento crea un pedido duplicado. Las escrituras se reintentan con una clave de idempotencia o comprobando antes dentro de la transacción.

## Cómo encaja con el resto del stack

`Microsoft.Data.SqlClient` es la capa de transporte, y casi nunca la capa con la que escribes el día a día:

| Herramienta | Qué aporta encima | Cuándo elegirla |
|---|---|---|
| `SqlCommand` en crudo | Nada: control total | `SqlBulkCopy`, varios *result sets*, control fino de tipos, diagnóstico |
| [Dapper](Dapper.md) | Mapea filas a objetos; el SQL lo escribes tú | Consultas de lectura y SQL a medida |
| [Entity Framework Core](Entity-Framework-Core.md) | Genera el SQL, sigue los cambios, migra el esquema | CRUD, agregados, [migraciones](../migraciones-de-esquema/README.md) |

Los tres comparten el mismo driver, la misma cadena de conexión y el mismo pool. Eso significa dos cosas útiles: puedes mezclarlos en la misma aplicación sin penalización, y todo lo de esta guía —`Encrypt`, `Max Pool Size`, errores transitorios— aplica igual aunque nunca escribas un `SqlCommand`.

**Cuándo NO usarlo en crudo.** No mapea objetos, no genera SQL, no gestiona migraciones ni valida que las tablas sean las que tu código espera, y no hay que pedírselo. Si te descubres escribiendo a mano el mapeo de veinte columnas a una clase, o construyendo SQL por concatenación para un filtro opcional, estás reimplementando peor lo que Dapper hace en una línea. El acceso crudo es para lo que los ORM no cubren, no para el CRUD de una aplicación entera.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `Login failed for user 'tienda_app'. (Error 18456)` | Credenciales mal, o el usuario no tiene acceso a esa base; el servidor nunca dice cuál de las dos, a propósito |
| `The certificate chain was issued by an authority that is not trusted` | `Encrypt=True` por defecto desde la 4.0 contra un certificado autofirmado |
| `A network-related or instance-specific error occurred... (provider: Named Pipes Provider, error: 40)` | Nombre de servidor mal, instancia parada, o TCP/IP deshabilitado en el SQL Server |
| `Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool.` | Pool agotado: conexiones sin `Dispose`, consultas lentas, o un pool por cadena interpolada |
| `Timeout expired` sin la parte del pool, tras 30 s | `Command Timeout`: la consulta es lenta o está esperando un bloqueo. Error número `-2` |
| `The connection does not support MultipleActiveResultSets.` | Dos lectores abiertos a la vez sobre la misma conexión: casi siempre una consulta dentro de un `while (reader.Read())` |
| `Invalid object name 'Productos'.` | Falta `Database=` en la cadena, o la tabla está en otro esquema y no pusiste `dbo.Productos` |
| `Must declare the scalar variable "@id".` | El nombre del parámetro no coincide con el del SQL, o falta el `Add` |
| `Parameterized Query ... expects the parameter '@nota', which was not supplied.` | `AddWithValue` con un `null` de C#: hay que pasar `DBNull.Value` |
| Consulta rápida en SSMS, lenta desde la aplicación | *Parameter sniffing* o `CONVERT_IMPLICIT` por `AddWithValue`: el plan de la aplicación no es el que ves en SSMS |
| CPU baja, base ociosa y respuestas de segundos | *Thread pool starvation* por `.Result`/`.Wait()` sobre operaciones del driver |
| `Cannot open database "TiendaDb" requested by the login. (Error 4060)` | Nombre de la base mal escrito o sin permisos: no es transitorio, no lo reintentes |

## Buenas prácticas avanzadas

- **Pon un `Application Name` distinto por servicio y aprovecha el `ClientConnectionId`.** `Application Name=Tienda.Api` hace que `sys.dm_exec_sessions` te diga qué servicio lanzó la consulta que bloquea la tabla; sin él, todo son sesiones anónimas. Y cuando falle una conexión, `SqlException.ClientConnectionId` te da el GUID exacto con el que correlacionar el fallo con los *extended events* del servidor. Registrarlo en el log es la diferencia entre depurar un incidente en diez minutos o en dos días.
- **Dimensiona `Max Pool Size` desde el servidor hacia atrás, no desde la aplicación hacia adelante.** Con seis instancias y el defecto de 100, pides hasta 600 conexiones a un servidor que probablemente rinda mejor con 150. Calcula el techo que soporta el motor, divídelo entre las instancias, y baja también `Connect Timeout` a 5 s: si el pool está agotado, esperar 15 s por petición solo alarga la cola. El indicador que hay que vigilar es el contador `NumberOfActiveConnections` frente a `NumberOfFreeConnections` de la fuente de eventos `Microsoft.Data.SqlClient.EventSource`.
- **Declara el tipo y el tamaño de cada parámetro, y no por manía.** Un `nvarchar(4000)` inferido contra una columna `varchar(50)` mete un `CONVERT_IMPLICIT` en el predicado y tumba el *index seek*; un tamaño variable según el valor genera un plan por longitud y ensucia la caché. Es el error de rendimiento más común del acceso a datos en .NET y el que menos se sospecha, porque el código es correcto y la consulta devuelve lo que debe.
- **`TrustServerCertificate=True` fuera de desarrollo es una decisión, no un descuido.** Cifra el canal pero no valida el interlocutor, así que el atacante en medio de la red descifra el tráfico y el login sin dejar rastro. Trátalo como lo que es: un valor que solo debe existir en la cadena de conexión local. Búscalo en las cadenas de producción como parte de la revisión de despliegue, y si el servidor lo soporta, migra a `Encrypt=Strict`, que lo ignora por completo.
- **Reintenta lecturas alegremente y escrituras nunca sin idempotencia.** `SqlException.IsTransient` te dice si el error merece otro intento, pero no si repetir la operación es seguro: un `-2` en un `INSERT` puede significar que la fila se escribió y la confirmación se perdió. Para escrituras, o hay una clave de idempotencia que el segundo intento choque contra un `UNIQUE`, o hay una comprobación previa dentro de la transacción. Y siempre con *backoff* exponencial y *jitter*: mil clientes reintentando a la vez rematan al servidor que se estaba levantando.
- **No dejes que el `CancellationToken` se pierda en el camino.** Pasarlo hasta `ExecuteReaderAsync` hace que el driver mande un `ATTENTION` al servidor y aborte la consulta cuando el cliente HTTP se va. Sin eso, un usuario impaciente que recarga cinco veces deja cinco consultas caras corriendo y cinco conexiones del pool ocupadas para nadie — que es exactamente cómo un pico de tráfico se convierte en una caída.

## Recursos didácticos

- [Referencia de palabras clave de la cadena de conexión](https://learn.microsoft.com/dotnet/api/microsoft.data.sqlclient.sqlconnection.connectionstring) — la lista completa con los valores por defecto y los sinónimos de cada palabra clave. Es la única fuente fiable para saber qué vale `Encrypt` en tu versión concreta.
- [connectionstrings.com](https://www.connectionstrings.com/sql-server/) — recetas de cadena de conexión para cada escenario (instancia nombrada, LocalDB, Entra ID, *failover partner*). Útil para copiar la forma y luego repasar cada parámetro con la tabla de esta guía.
- [Repositorio de Microsoft.Data.SqlClient en GitHub](https://github.com/dotnet/SqlClient) — el código, las *issues* y —lo más valioso— las notas de cada versión. Buscar un mensaje de error literal en las *issues* resuelve más rápido que cualquier búsqueda genérica.
- [Notas de la versión 4.0](https://github.com/dotnet/SqlClient/blob/main/release-notes/4.0/4.0.0.md) — el documento que explica el cambio de `Encrypt` a `True` por defecto y sus implicaciones. Lectura obligada antes de subir de la 3.x a la 4.x o superior.
- [Documentación de Polly](https://www.pollydocs.org/) — patrones de reintento, *circuit breaker* y *timeout* con el vocabulario de .NET, y ejemplos de cómo combinarlos en una sola tubería de resiliencia.

---

*En resumen: `Microsoft.Data.SqlClient` es el cable entre tu código C# y SQL Server — y casi todo lo que se rompe en la capa de datos de un backend .NET está en tres sitios suyos: el `Encrypt=True` por defecto, el pool que se agota porque las consultas son lentas, y el `AddWithValue` que tumba el índice sin decir nada.*
