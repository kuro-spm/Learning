# Limitar intentos de login

## ¿Qué es?

Limitar los intentos de login es poner un tope a cuántas veces se puede probar una contraseña contra un endpoint de autenticación, por origen (IP) y por cuenta (email), de modo que adivinar credenciales a base de probar deje de ser viable.

## ¿Por qué existe?

Un formulario de login es una pregunta que cualquiera puede repetir millones de veces: «¿es esta la contraseña de `ana@tienda.com`?». Sin límite, quien ataca solo necesita tiempo y una lista de candidatas. Hay tres formas habituales de explotarlo:

- **Fuerza bruta**: probar muchas contraseñas contra una misma cuenta.
- **Credential stuffing**: probar pares email/contraseña filtrados de otras webs, aprovechando que mucha gente reutiliza contraseña. Cada cuenta recibe pocos intentos, pero desde miles de IPs.
- **Denegación de servicio por coste**: un login válido ejecuta una [función de derivación de claves](../algoritmos-de-hash/Funciones-De-Derivacion-De-Claves.md) deliberadamente cara (bcrypt, Argon2...). Esa lentitud protege los hashes robados, pero también significa que cada intento cuesta CPU y memoria: mil intentos por segundo, aunque todos fallen, pueden tumbar el servidor.

> Si ya conoces los cajeros automáticos, piensa en el límite de intentos como el «tres PIN erróneos y se retiene la tarjeta»: no hace que adivinar el PIN sea imposible, hace que no merezca la pena intentarlo.

## ¿Cuándo y para qué se usa?

En cualquier endpoint que compruebe una credencial: login, recuperación de contraseña, verificación de códigos de un solo uso (2FA), cambio de contraseña. El ejemplo conductor de esta guía es una tienda online con el endpoint `POST /api/auth/login`, que recibe `{ "email": "...", "password": "..." }`.

Hay dos límites complementarios, y ninguno sustituye al otro:

| Límite | Frena | No frena |
|---|---|---|
| **Por IP** | Un origen que martillea el endpoint (fuerza bruta simple, DoS por KDF) | Un ataque repartido entre muchas IPs |
| **Por cuenta** | Cualquier número de IPs apuntando a la misma cuenta | Muchas cuentas con pocos intentos cada una (credential stuffing «de baja intensidad») |

Los dos van **dentro de la aplicación**, porque ahí se sabe si un intento falló. Un firewall o [Fail2ban](../../devops/despliegue-en-vps/Fail2ban.md) actúan por encima, leyendo logs o conexiones, y se combinan bien con ellos, pero no los reemplazan.

---

## Primer límite: por IP con el `RateLimiter` de ASP.NET Core

Desde .NET 7, ASP.NET Core incluye un middleware de *rate limiting* sin paquetes adicionales. Se define una **política** con nombre y se aplica a los endpoints que interese. Este es un ejemplo completo y funcional: 10 intentos por minuto y por IP.

```csharp
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    options.AddPolicy("auth-login", httpContext =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: httpContext.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 10,                    // peticiones permitidas por ventana
                Window = TimeSpan.FromMinutes(1),    // duración de la ventana
                QueueLimit = 0                       // sin cola: la que sobra se rechaza
            }));
});

var app = builder.Build();

app.UseRouting();
app.UseRateLimiter();   // después de UseRouting para que vea los metadatos del endpoint

app.MapPost("/api/auth/login", (LoginRequest request) => Results.Ok())
   .RequireRateLimiting("auth-login");

app.Run();

record LoginRequest(string Email, string Password);
```

Las piezas:

- **`AddPolicy("auth-login", ...)`** registra una política con nombre. El delegado recibe el `HttpContext` y devuelve a qué **partición** pertenece la petición.
- **Partición**: cada clave distinta (aquí, cada IP) tiene su propio contador. `GetFixedWindowLimiter` crea un limitador de **ventana fija** por clave: 10 peticiones cada minuto, y al empezar la siguiente ventana el contador vuelve a cero.
- **`RejectionStatusCode`** es el código de las peticiones rechazadas. Por defecto es `503`; para un límite de tasa lo correcto es `429 Too Many Requests`.
- **`RequireRateLimiting("auth-login")`** (minimal APIs) o `[EnableRateLimiting("auth-login")]` (controllers) activan la política en ese endpoint. El resto de la API no se limita.

Con la petición número 11 dentro del mismo minuto, la respuesta es:

```
HTTP/1.1 429 Too Many Requests
```

Si el cliente necesita un cuerpo con formato propio (por ejemplo, el mismo JSON de error que el resto de la API), se define `OnRejected`:

```csharp
options.OnRejected = async (context, cancellationToken) =>
{
    context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
    await context.HttpContext.Response.WriteAsJsonAsync(
        new { code = "TOO_MANY_REQUESTS", message = "Demasiados intentos. Inténtalo más tarde." },
        cancellationToken);
};
```

Tres detalles que cuestan un rato descubrir:

- **El orden de middlewares importa.** `UseRateLimiter` va después de `UseRouting` y **antes** de `UseAuthentication`, para rechazar sin pagar la validación de sesión. Si hay CORS, `UseCors` debe ir antes: si no, las respuestas 429 salen sin cabeceras CORS y el navegador no deja leer su cuerpo.
- **La IP debe ser la real.** Detrás de un reverse proxy, `RemoteIpAddress` es la del proxy: todos los usuarios compartirían contador y el límite bloquearía a media tienda. Cómo resolverlo se explica en [IP real detrás de un reverse proxy](IP-real-detras-de-un-reverse-proxy.md).
- **La ventana fija tiene un efecto borde.** Quien envía 10 peticiones al final de una ventana y 10 al principio de la siguiente consigue 20 en pocos segundos. Si importa, existe `GetSlidingWindowLimiter`, que reparte la ventana en segmentos; para un login, la ventana fija suele bastar.

## El límite por IP no basta: ataques distribuidos

Una botnet con 5 000 IPs puede enviar 10 intentos por minuto desde cada una: 50 000 intentos por minuto contra `ana@tienda.com` sin que ningún límite por IP salte. Y el credential stuffing es el caso opuesto: pocas peticiones por IP, pero contra miles de cuentas.

La respuesta es un segundo contador cuya clave no sea el origen sino el **objetivo**: la cuenta.

## Segundo límite: por cuenta, con backoff exponencial

La idea: por cada email se guardan los **fallos consecutivos**. Al llegar a un umbral, la cuenta queda bloqueada un tiempo que se **duplica** con cada fallo adicional, hasta un tope. Un login correcto lo borra todo.

```csharp
using System.Collections.Concurrent;

public sealed class LoginThrottleSettings
{
    public int UmbralFallos { get; set; } = 5;                                // fallos antes de bloquear
    public TimeSpan BloqueoBase { get; set; } = TimeSpan.FromMinutes(1);      // primer bloqueo
    public TimeSpan BloqueoMaximo { get; set; } = TimeSpan.FromMinutes(30);   // tope
    public TimeSpan VentanaOlvido { get; set; } = TimeSpan.FromMinutes(15);   // sin fallos en este tiempo = se olvida
}

public sealed class LoginThrottle(LoginThrottleSettings cfg, TimeProvider reloj)
{
    private sealed record Estado(int Fallos, DateTimeOffset? BloqueoHasta, DateTimeOffset UltimoFallo);

    private readonly ConcurrentDictionary<string, Estado> _estado = new();

    public bool EstaBloqueada(string email) =>
        _estado.TryGetValue(Normalizar(email), out var e)
        && e.BloqueoHasta is { } hasta && hasta > reloj.GetUtcNow();

    public void RegistrarFallo(string email)
    {
        var ahora = reloj.GetUtcNow();
        _estado.AddOrUpdate(
            Normalizar(email),
            _ => TrasFallo(new Estado(0, null, ahora), ahora),
            (_, actual) =>
            {
                // Ventana de olvido: si hace mucho del último fallo, se empieza de cero.
                if (ahora - actual.UltimoFallo > cfg.VentanaOlvido)
                    actual = new Estado(0, null, ahora);
                return TrasFallo(actual, ahora);
            });
    }

    public void Resetear(string email) => _estado.TryRemove(Normalizar(email), out _);

    private Estado TrasFallo(Estado actual, DateTimeOffset ahora)
    {
        var fallos = actual.Fallos + 1;
        DateTimeOffset? bloqueoHasta = null;

        if (fallos >= cfg.UmbralFallos)
        {
            // BloqueoBase * 2^(fallos - umbral), con tope. El exponente se acota para que 2^n no desborde.
            var exponente = Math.Min(fallos - cfg.UmbralFallos, 30);
            var ticks = cfg.BloqueoBase.Ticks * Math.Pow(2, exponente);
            var duracion = ticks >= cfg.BloqueoMaximo.Ticks
                ? cfg.BloqueoMaximo
                : TimeSpan.FromTicks((long)ticks);
            bloqueoHasta = ahora + duracion;
        }
        return new Estado(fallos, bloqueoHasta, ahora);
    }

    private static string Normalizar(string email) => (email ?? "").Trim().ToLowerInvariant();
}
```

Con `UmbralFallos = 5` y `BloqueoBase = 1 min`, la progresión es:

| Fallo consecutivo | Resultado |
|---|---|
| 1 a 4 | Sin bloqueo |
| 5 | Bloqueo de 1 min |
| 6 | Bloqueo de 2 min |
| 7 | Bloqueo de 4 min |
| 8 | Bloqueo de 8 min |
| 9 | Bloqueo de 16 min |
| 10 o más | Bloqueo de 30 min (el tope) |

Cada decisión de diseño tiene su motivo:

- **Normalizar el email** (`Trim` + `ToLowerInvariant`) evita que `Ana@Tienda.com` y `ana@tienda.com` tengan contadores distintos y multipliquen los intentos.
- **Fallos consecutivos y ventana de olvido.** Quien se equivoca una vez al mes no debe acumular: si pasan más de 15 minutos sin fallos, el contador se reinicia. Es lo que permite que una persona legítima recupere el acceso sin intervención.
- **Backoff exponencial con tope.** Frena cada vez más a quien insiste, pero el tope impide que un atacante bloquee una cuenta durante días (el bloqueo en sí es un arma, ver más abajo).
- **Reset al login correcto.** Se llama a `Resetear` cuando la contraseña es válida; si no, quien acierta a la sexta seguiría arrastrando fallos.
- **El delegado de `AddOrUpdate` debe ser puro.** `ConcurrentDictionary` puede reejecutarlo si hay contención, así que no debe tener efectos secundarios (nada de logging ni contadores dentro).

### Cómo se usa en el endpoint

```csharp
app.MapPost("/api/auth/login", async (LoginRequest req, LoginThrottle throttle, IUserService users) =>
{
    if (throttle.EstaBloqueada(req.Email))
        return Results.Json(new { code = "TOO_MANY_REQUESTS", message = "Demasiados intentos. Inténtalo más tarde." },
                            statusCode: StatusCodes.Status429TooManyRequests);

    var user = await users.ValidarCredencialesAsync(req.Email, req.Password);
    if (user is null)
    {
        throttle.RegistrarFallo(req.Email);
        return Results.Json(new { code = "INVALID_CREDENTIALS", message = "Email o contraseña incorrectos." },
                            statusCode: StatusCodes.Status401Unauthorized);
    }

    throttle.Resetear(req.Email);
    return Results.Ok(new { token = "..." });
}).RequireRateLimiting("auth-login");
```

Se comprueba el bloqueo **antes** de validar la contraseña: así una cuenta bloqueada no gasta CPU en el KDF, y quien ataca no puede seguir probando contraseñas durante el bloqueo (acertar la correcta tampoco da acceso mientras dure).

### Respuestas que no filtran información

El bloqueo por cuenta introduce dos fugas fáciles de cometer:

- **Revelar el tiempo restante** (`"Cuenta bloqueada, vuelve en 3 min"` o una cabecera `Retry-After` por cuenta) indica exactamente cuándo reintentar y confirma que hay fallos acumulados.
- **Revelar que la cuenta existe.** Si solo las cuentas reales se bloquean, o el mensaje de bloqueo es distinto del de «email o contraseña incorrectos», se puede enumerar qué emails están registrados.

Por eso la respuesta de bloqueo es **genérica** (un 429 con mensaje fijo) y el contador **registra también los fallos de emails que no existen**: probar `inventado@x.com` acumula fallos y acaba en el mismo 429 que atacar una cuenta real. Es justo esto lo que provoca el problema siguiente.

> El bloqueo por cuenta es además una vía de DoS dirigido: quien conoce un email puede bloquear a su titular a base de fallos. Por eso el bloqueo es **temporal y con tope**, y la ventana de olvido garantiza que se recupera solo. Para cuentas críticas se complementa con avisos por email o un desbloqueo por enlace.

## El problema de la memoria sin cota

El `ConcurrentDictionary` del ejemplo tiene un fallo grave: **la clave la controla quien ataca**. Como también se registran fallos de emails inexistentes, cada intento con un email inventado crea una entrada nueva y nada la elimina. Un script que pruebe `a1@x.com`, `a2@x.com`, `a3@x.com`... convierte el diccionario en una fuga de memoria.

La ventana de olvido **no lo arregla**: solo reinicia el contador cuando la misma clave vuelve a aparecer, pero la entrada de un email que nunca regresa se queda para siempre. Hace falta un barrido que elimine las entradas muertas.

### Qué se puede eliminar

Una entrada se puede borrar cuando ya no aporta nada, es decir, cuando se cumplen **las dos** condiciones:

1. **Está olvidada**: `ahora - UltimoFallo > VentanaOlvido`. Si volviera ese email, el contador empezaría de cero igualmente.
2. **No tiene bloqueo vigente**: `BloqueoHasta` es `null` o ya pasó. Hay un caso trampa: con bloqueos largos (30 min) y ventana de olvido corta (15 min), una entrada puede estar «olvidada» y seguir bloqueada. Borrarla desbloquearía la cuenta antes de tiempo.

### El barrido: `BackgroundService` + `PeriodicTimer`

```csharp
public sealed class BarridoLoginThrottleWorker(
    LoginThrottle throttle,
    ILogger<BarridoLoginThrottleWorker> logger) : BackgroundService
{
    private static readonly TimeSpan Intervalo = TimeSpan.FromMinutes(5);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(Intervalo);

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                var eliminadas = throttle.PurgarExpiradas();
                logger.LogInformation("Barrido del throttle: {Eliminadas} entradas eliminadas", eliminadas);
            }
            catch (Exception ex)
            {
                // Un fallo puntual no debe tumbar el worker: se reintenta en el siguiente tick.
                logger.LogError(ex, "Fallo al barrer el throttle de login");
            }
        }
    }
}
```

Se registra con `AddSingleton<LoginThrottle>()` y `AddHostedService<BarridoLoginThrottleWorker>()`. Importa que `LoginThrottle` sea **singleton**: si fuera *scoped*, cada petición tendría su propio diccionario vacío y el límite no limitaría nada.

`PeriodicTimer` (.NET 6) es la forma moderna de un bucle periódico en un `BackgroundService`: `WaitForNextTickAsync` espera de forma asíncrona y lanza `OperationCanceledException` al cancelar el token, lo que cierra el bucle limpiamente al apagar la aplicación. Además espera el intervalo completo antes del primer barrido (no barre al arrancar). El intervalo solo afecta a **cuánta memoria muerta se tolera**, no a la corrección, así que 5 minutos son razonables.

### La carrera: por qué `TryRemove` con el par clave-valor

Ahora el método del barrido. La versión ingenua tiene un fallo sutil:

```csharp
// ❌ Pierde fallos: lee, decide y borra en pasos no atómicos
foreach (var entrada in _estado)
{
    if (EstaMuerta(entrada.Value, ahora))
        _estado.TryRemove(entrada.Key, out _);   // borra lo que haya AHORA bajo esa clave
}
```

Entre que el barrido lee el estado y borra, otro hilo puede registrar un fallo en esa misma clave:

```
Barrido                                    Hilo del login
-------                                    --------------
1. Lee Estado A (último fallo hace 20 min,
   olvidado, sin bloqueo): candidato a borrar
                                           2. RegistrarFallo: AddOrUpdate sustituye
                                              A por Estado B (1 fallo, de ahora)
3. TryRemove(clave): borra lo que hay
   bajo la clave, que ya es B
                                           → el fallo del paso 2 se ha perdido
```

La solución es la sobrecarga que recibe el **par** clave-valor y solo elimina si el valor actual sigue siendo el que se evaluó:

```csharp
public int PurgarExpiradas()
{
    var ahora = reloj.GetUtcNow();
    var eliminadas = 0;

    // Enumerar un ConcurrentDictionary es seguro con escrituras concurrentes: no lanza excepción.
    foreach (var entrada in _estado)
    {
        var e = entrada.Value;
        var olvidada = ahora - e.UltimoFallo > cfg.VentanaOlvido;
        var sinBloqueoVigente = e.BloqueoHasta is not { } hasta || hasta <= ahora;

        // TryRemove(KeyValuePair) elimina solo si el valor sigue siendo el MISMO que se evaluó.
        if (olvidada && sinBloqueoVigente && _estado.TryRemove(entrada))
            eliminadas++;
    }

    return eliminadas;
}
```

Con `TryRemove(entrada)` el paso 3 del diagrama **falla** (el valor ya es B, distinto de A) y el fallo recién registrado sobrevive. Si el orden es el contrario (el barrido borra primero y `AddOrUpdate` crea después una entrada nueva), el resultado también es correcto: la entrada borrada estaba olvidada y el fallo nuevo empieza un contador limpio.

Matiz: la comparación del valor usa `EqualityComparer<TValue>.Default`. Al ser `Estado` un `record`, la igualdad es **por valor** (campo a campo), no por referencia. Funciona aquí porque cualquier fallo nuevo cambia `UltimoFallo`; con un tipo cuyos campos pudieran repetirse exactamente, habría que añadir un número de versión.

### Alternativa: tope duro con LRU

El barrido limita la memoria **en el tiempo**, no **en el número**: durante los 5 minutos entre barridos, un atacante puede insertar muchísimas claves. Si eso es un riesgo real (API expuesta sin filtro previo), la alternativa es un **tope duro**: una caché con máximo de entradas y expulsión de la menos usada recientemente (LRU), como `MemoryCache` con `SizeLimit` o una estructura propia.

La contrapartida: bajo ataque, el LRU **expulsa entradas legítimas** (incluidas cuentas bloqueadas) para hacer sitio a las inventadas, y quien ataca consigue «desbloquear» su objetivo inundando de claves basura. Por eso conviene el barrido cuando el límite por IP ya frena el volumen de inserciones (10 por minuto y por IP obligan a usar muchas IPs para llenar la memoria), y el tope duro cuando hace falta una garantía de memoria incluso en el peor caso. Ambos se pueden combinar.

## Limitaciones del estado en memoria

El diccionario vive en el proceso, y eso tiene dos consecuencias:

- **Se pierde al reiniciar.** Un despliegue o un *crash* devuelve el contador a cero. Para un control de corta duración es aceptable (el límite por IP sigue en pie), pero quien pudiera provocar reinicios los usaría para saltarse el bloqueo.
- **No se comparte entre instancias.** Con dos réplicas detrás de un balanceador, cada una tiene su propio contador: el umbral efectivo se multiplica por el número de instancias. En ese caso el estado debe moverse a un almacén compartido: **Redis** es la opción habitual (contador con `INCR` y expiración con `EXPIRE`, y la limpieza la hace Redis sin barrido propio), o una tabla en base de datos si el volumen es bajo. El `RateLimiter` de ASP.NET Core por IP tiene la misma limitación: es en memoria y por instancia.

## Cómo testearlo

El tiempo es el enemigo de los tests: probar que un bloqueo expira a los 15 minutos no puede costar 15 minutos. La solución es **no leer el reloj directamente** (`DateTime.UtcNow`) sino inyectar `TimeProvider` (.NET 8), como hace el ejemplo, y usar en los tests un reloj falso: `FakeTimeProvider`, del paquete `Microsoft.Extensions.TimeProvider.Testing`, cuyo `Advance(TimeSpan)` mueve el tiempo a voluntad.

```csharp
using Microsoft.Extensions.Time.Testing;

public class LoginThrottleTests
{
    private static (LoginThrottle Sut, FakeTimeProvider Reloj) Crear()
    {
        var reloj = new FakeTimeProvider(new DateTimeOffset(2026, 1, 1, 10, 0, 0, TimeSpan.Zero));
        return (new LoginThrottle(new LoginThrottleSettings(), reloj), reloj);
    }

    [Fact]
    public void TrasElUmbral_LaCuentaQuedaBloqueadaYSeDesbloqueaConElTiempo()
    {
        var (sut, reloj) = Crear();
        for (var i = 0; i < 5; i++) sut.RegistrarFallo("ana@tienda.com");

        Assert.True(sut.EstaBloqueada("ana@tienda.com"));

        reloj.Advance(TimeSpan.FromMinutes(2));   // más que el bloqueo base de 1 min
        Assert.False(sut.EstaBloqueada("ana@tienda.com"));
    }

    [Fact]
    public void Barrido_EliminaEntradasOlvidadas()
    {
        var (sut, reloj) = Crear();
        sut.RegistrarFallo("vieja@tienda.com");
        reloj.Advance(TimeSpan.FromMinutes(16));   // fuera de la ventana de olvido (15 min)

        Assert.Equal(1, sut.PurgarExpiradas());
    }
}
```

La parte más delicada es la carrera. No se puede probar con un único intento, porque casi siempre sale bien por azar: se repite muchas veces y se comprueba una **invariante**. Pase lo que pase con el orden de los hilos, tras ejecutar barrido y registro de fallo en paralelo debe haber quedado registrado exactamente un fallo.

```csharp
[Fact]
public async Task BarridoConcurrenteConRegistrarFallo_NoPierdeElFalloRecienRegistrado()
{
    for (var i = 0; i < 500; i++)
    {
        var reloj = new FakeTimeProvider(new DateTimeOffset(2026, 1, 1, 10, 0, 0, TimeSpan.Zero));
        var sut = new LoginThrottle(new LoginThrottleSettings { UmbralFallos = 2 }, reloj);

        sut.RegistrarFallo("ana@tienda.com");
        reloj.Advance(TimeSpan.FromMinutes(16));        // entrada expirada: candidata al barrido

        var barrido = Task.Run(() => sut.PurgarExpiradas());
        var fallo = Task.Run(() => sut.RegistrarFallo("ana@tienda.com"));
        await Task.WhenAll(barrido, fallo);

        // Con umbral 2, un fallo más debe bloquear. Si el barrido se hubiera
        // llevado por delante el fallo recién registrado, no bloquearía.
        sut.RegistrarFallo("ana@tienda.com");
        Assert.True(sut.EstaBloqueada("ana@tienda.com"), $"Iteración {i}: se perdió un fallo");
    }
}
```

Este es el test que delata la versión ingenua con `TryRemove(clave, out _)`: con el par clave-valor pasa siempre; con la clave sola, puede fallar de forma intermitente en alguna de las vueltas.

## Buenas prácticas avanzadas

- **Limita por IP y por cuenta a la vez.** El límite por IP es barato y va primero (antes de autenticar); el de cuenta necesita saber qué falló. Quitar uno deja abierto justo el ataque que el otro no cubre.
- **Haz que respuesta y tiempo sean iguales exista o no la cuenta.** Un 401 genérico no sirve si, para emails inexistentes, el servidor responde en 2 ms y para los reales en 300 ms (el KDF): se enumera por latencia. Ejecuta un hash de relleno cuando el usuario no exista.
- **Si registras fallos de claves arbitrarias, el barrido deja de ser opcional.** Contar también los emails inexistentes evita la enumeración, y es justo lo que abre la fuga de memoria. Cualquier estructura indexada por algo que controla quien ataca necesita expiración o tope.
- **No registres el email en los logs del bloqueo.** El número de fallos y el fin del bloqueo bastan para detectar un ataque; el email filtra datos personales y, a menudo, contraseñas escritas en el campo equivocado.
- **Trata el bloqueo como vector de DoS.** Tope máximo, ventana de olvido y, para cuentas críticas, un camino de recuperación fuera de banda. Un bloqueo sin tope convierte un límite de seguridad en una herramienta para dejar fuera a cualquiera.
- **No confíes en `X-Forwarded-For` sin lista de proxies conocidos.** Esa cabecera la escribe quien quiera: si la aplicación la acepta de cualquiera, un atacante cambia de IP en cada petición y esquiva el límite por IP por completo.

## Documentación oficial

- [Rate limiting middleware en ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit) — los algoritmos disponibles (fixed window, sliding window, token bucket, concurrency), particionado y `OnRejected`. Empieza por la sección de ventana fija y la de políticas con nombre.
- [Authentication Cheat Sheet de OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) — protección frente a ataques automatizados y mensajes de error genéricos; es la referencia de qué se espera de un login resistente.
- [Credential Stuffing Prevention Cheat Sheet de OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Credential_Stuffing_Prevention_Cheat_Sheet.html) — defensas contra el ataque distribuido que el límite por IP no ve (MFA, comprobación de contraseñas filtradas, huella del dispositivo).
- [Introducción a `TimeProvider`](https://learn.microsoft.com/en-us/dotnet/standard/datetime/timeprovider-overview) — cómo abstraer el reloj y sustituirlo en tests.

## Recursos didácticos

- [Have I Been Pwned: Pwned Passwords](https://haveibeenpwned.com/Passwords) — permite comprobar si una contraseña aparece en filtraciones conocidas; da una idea intuitiva de por qué el credential stuffing funciona. Para entender qué se guarda de una contraseña y qué es una sesión, véase [Contraseñas vs tokens de sesión](../algoritmos-de-hash/Contrasenas-Vs-Tokens-De-Sesion.md).

---

*En resumen: limita por IP para frenar a quien martillea y por cuenta para frenar a quien se reparte, responde siempre con el mismo 429 genérico, y recuerda que cualquier contador indexado por algo que controla quien ataca necesita un barrido que lo vacíe.*
