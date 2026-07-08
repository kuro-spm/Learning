# CancellationToken

## ¿Qué es?

`CancellationToken` es el mecanismo de .NET para pedir, de forma cooperativa, que una operación asíncrona en marcha se detenga, y para propagar esa petición a través de toda una cadena de llamadas `async`.

## ¿Por qué existe?

Cuando un cliente cancela una petición HTTP (cierra la pestaña, se le acaba el tiempo de espera) o el propio proceso se está apagando, sin un mecanismo de cancelación el servidor sigue haciendo un trabajo que ya nadie va a leer: terminar una consulta lenta a base de datos, esperar la respuesta de una API externa cuyo resultado se va a descartar. .NET no puede simplemente "matar" un hilo a mitad de una operación de forma segura, así que la cancelación es **cooperativa**: cada paso de la cadena tiene que comprobar por su cuenta si le han pedido parar, y para eso necesita una señal que le llegue.

> Piensa en el testigo de una carrera de relevos: cada método `async` recibe el `CancellationToken` como quien recibe el testigo, y lo pasa a la siguiente llamada `async` que también sabe leerlo. Si alguien en la cadena se queda el testigo y no lo pasa (o no lo mira), el aviso de "parad" no llega más allá de ese punto.

## ¿Cuándo y para qué se usa?

En cualquier método `async` que haga E/S y pueda tardar: una consulta a base de datos, una llamada HTTP, la lectura de un fichero grande. Es especialmente relevante en los controladores de ASP.NET Core, que reciben un token ya vinculado a la conexión del cliente; en bucles que procesan lotes largos; y en cualquier llamada a un servicio externo que soporte cancelación.

## Lo mínimo que necesitas saber

**1. El controlador recibe el token, no lo crea**

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> ObtenerPedido(int id, CancellationToken ct)
{
    var pedido = await repositorio.ObtenerPorIdAsync(id, ct);
    return Ok(pedido);
}
```

ASP.NET Core vincula automáticamente ese `CancellationToken` a la conexión HTTP: si el cliente se desconecta, el token se cancela solo.

**2. Cada método `async` lo acepta y lo reenvía, sin transformarlo**

```csharp
public async Task<Pedido> ObtenerPorIdAsync(int id, CancellationToken ct)
{
    return await connection.QuerySingleAsync<Pedido>(
        "SELECT * FROM Pedidos WHERE Id = @id", new { id }, cancellationToken: ct);
}
```

La convención habitual es que toda firma pública `async` termine con `CancellationToken ct = default`, y que ese mismo token viaje sin cambios desde el punto de entrada hasta la llamada final (Dapper, `HttpClient`, etc.).

**3. Comprobarlo explícitamente en bucles largos sin llamadas `async`**

```csharp
foreach (var lote in lotes)
{
    ct.ThrowIfCancellationRequested();
    ProcesarLote(lote);
}
```

**4. Crear un token propio con límite de tiempo**

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await clienteHttp.GetAsync(url, cts.Token);
```

**5. La cancelación se manifiesta como una excepción, que no es un error real**

```csharp
try
{
    await OperacionLarga(ct);
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // el cliente se desconectó; no es un fallo del sistema, no hace falta loguearlo como error
}
```

## Lo que NO hace

- **No detiene nada a la fuerza** — es cooperativo: si un método no comprueba el token ni lo pasa a algo que lo respete, sigue ejecutándose hasta el final igualmente.
- **No deshace lo que ya se hizo** — si a mitad de una operación ya se escribió algo en base de datos, cancelar el token no revierte esa escritura.
- **No sustituye a un timeout de conexión HTTP o de base de datos** — son señales distintas que suelen combinarse, no una la reemplaza a la otra.

## Buenas prácticas avanzadas

- **Propágalo de extremo a extremo, sin atajos "por comodidad"** — sustituir el token recibido por `CancellationToken.None` en algún punto intermedio de la cadena (porque "aquí total no hace falta") rompe la cancelación para todo lo que se ejecuta a partir de ahí, aunque el resto de la cadena lo siga propagando correctamente.
- **No trates `OperationCanceledException` como un error de negocio** — capturarla y loguearla en nivel `Error` genera ruido y falsas alarmas cada vez que un usuario cierra una pestaña antes de tiempo; es una cancelación esperada, no un fallo del sistema.
- **Cuidado con atar al token de la petición un trabajo que debe sobrevivirla** — si tras devolver una respuesta HTTP arrancas un proceso en segundo plano (enviar un email, generar un informe) y sigue usando el `CancellationToken` de esa petición, se cancelará en cuanto el cliente reciba la respuesta y cierre la conexión, aunque ese trabajo debía continuar de forma independiente.
- **Combina tu propio timeout con el token entrante mediante `CancellationTokenSource.CreateLinkedTokenSource`** — así una llamada externa se cancela tanto si el cliente se desconecta como si tarda más de lo razonable, sin tener que elegir entre una señal y la otra.

---

*En resumen: `CancellationToken` es el testigo cooperativo que viaja de extremo a extremo por una cadena de llamadas asíncronas para avisar de que ya nadie espera el resultado — cada eslabón tiene que recibirlo, respetarlo y pasarlo, sin sustituirlo nunca por "no hace falta cancelar aquí".*
