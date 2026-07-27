# Credenciales en llamadas salientes

## ¿Qué es?

El conjunto de prácticas para que la credencial que tu servidor envía al llamar a una API externa — un token, una `api-key` — **no se filtre por el camino**: ni a los logs, ni a un host equivocado tras una redirección, ni a un mensaje de error, ni a un servidor que ha elegido un atacante.

## ¿Por qué existe?

Cuando tu backend actúa de **cliente** de otra API, deja de ser el que valida credenciales y pasa a ser el que las presenta. El secreto puede estar impecablemente guardado *en reposo* — en user-secrets, en un *vault*, en variables de entorno — y aun así escaparse **en el instante de usarlo**, por caminos que no saltan a la vista: un log que serializa la petición entera, una respuesta `302` que reenvía tus cabeceras a otro dominio, una excepción cuyo mensaje incluye la URL completa con la clave dentro.

La diferencia importa porque las dos cosas se protegen con herramientas distintas. Dónde vive el secreto lo cubre la colección de [gestión de secretos en desarrollo](../gestion-de-secretos-en-desarrollo/README.md). Esta ficha empieza donde aquella acaba: el secreto ya está cargado en memoria y hay que usarlo sin dejar rastro.

> Piensa en la clave de tu casa. Guardarla en una caja fuerte (reposo) no sirve de nada si luego la enseñas al pedir la hora por la calle, la dejas fotografiada en un ticket o se la das a quien te dice "dámela, ya se la llevo yo al cerrajero" (uso).

## ¿Cuándo y para qué se usa?

Siempre que tu servidor consuma una API externa autenticada. El ejemplo que recorre toda la ficha es una **tienda online en C#/.NET** que cobra los pedidos contra una pasarela de pago externa en `api.pasarela-pago.com`, usando una clave de API. El pedido de referencia es el **#4711**.

```csharp
// El escenario base: cobrar el pedido #4711 contra la pasarela
var respuesta = await client.PostAsJsonAsync("/v1/cargos", new
{
    pedido = "4711",
    importe = 89.90m,
    moneda = "EUR"
});
```

Una llamada así de sencilla tiene, como mínimo, seis puntos por los que la clave puede escaparse. Los vemos uno a uno, en orden de frecuencia real.

---

## 1. La credencial en la URL

Es el error más común y el más difícil de deshacer, porque la URL no es un dato privado: **se copia sola** a todas partes.

```csharp
// ❌ La clave viaja en la query string
var url = $"https://api.pasarela-pago.com/v1/cargos?api_key={clave}&pedido=4711";
var respuesta = await client.PostAsync(url, contenido);
```

```csharp
// ✅ La clave viaja en una cabecera; la URL no contiene nada sensible
client.DefaultRequestHeaders.Add("api-key", clave);
var respuesta = await client.PostAsJsonAsync(
    "https://api.pasarela-pago.com/v1/cargos",
    new { pedido = "4711", importe = 89.90m });
```

Ambas llamadas funcionan igual y devuelven el mismo `200 OK`. La diferencia está en dónde acaba la cadena `api_key=sk_live_...`:

| Destino | Qué pasa con la URL | Qué pasa con la cabecera |
|---|---|---|
| Log de acceso del servidor destino | Se registra entera, en claro | No se registra |
| Proxy corporativo o CDN intermedio | Queda en su registro de tráfico | No lo ve (va cifrada en TLS, igual que la URL, pero el proxy sí registra la línea de petición si hace de terminador) |
| Trazas de APM / logging del cliente | Se registra a nivel `Information` | Solo a nivel `Trace`, y redactable |
| Historial del navegador y cabecera `Referer` | Si esa URL llega alguna vez a un navegador, se propaga al siguiente sitio visitado | No aplica |
| Un ticket de soporte con "esta es la URL que falla" | La clave va dentro | No |

La cabecera no es mágica: viaja por el mismo canal. Lo que cambia es que **nada ni nadie la registra por defecto**, mientras que la URL se registra en todas partes por diseño.

## 2. Redirecciones: tu token acaba en un host que no elegiste

`HttpClient` sigue las redirecciones automáticamente (`AllowAutoRedirect = true` por defecto). Ante un `302` responde repitiendo la petición contra la nueva ubicación, y ahí está el problema: **reenvía tus cabeceras**.

.NET moderno hace media defensa. Al redirigir a un origen distinto (otro esquema, host o puerto), elimina la cabecera `Authorization` tipada. Pero **no elimina las cabeceras personalizadas**, que es exactamente cómo autentican la mayoría de pasarelas de pago:

| Cabecera | ¿Se reenvía a otro origen? |
|---|---|
| `Authorization: Bearer ...` | No — .NET la elimina |
| `api-key: sk_live_...` | **Sí** |
| `X-Api-Key`, `X-Auth-Token`, cualquier cabecera propia | **Sí** |

Es decir: en el escenario de esta ficha, con una `api-key`, la fuga es real. Basta con que `api.pasarela-pago.com` sufra un *open redirect* o con que alguien manipule la configuración del endpoint para que tu clave de producción aterrice en el log de un servidor ajeno.

```csharp
// ✅ Ningún cliente autenticado sigue redirecciones sin supervisión
var handler = new HttpClientHandler { AllowAutoRedirect = false };
using var client = new HttpClient(handler)
{
    BaseAddress = new Uri("https://api.pasarela-pago.com/"),
    Timeout = TimeSpan.FromSeconds(30)
};
client.DefaultRequestHeaders.Add("api-key", clave);

var respuesta = await client.PostAsJsonAsync("/v1/cargos", new { pedido = "4711" });

if ((int)respuesta.StatusCode is >= 300 and < 400)
{
    // Ahora la redirección es visible y decides tú si la sigues
    logger.LogWarning("La pasarela redirige a {Destino} para el pedido {Pedido}",
        respuesta.Headers.Location, "4711");
}
```

Con `AllowAutoRedirect = false`, el `302` deja de ser invisible: llega como respuesta normal, con su `Location`, y tu código decide. Si de verdad esperas redirecciones, valida el host de destino antes de repetir la llamada con la credencial.

## 3. Logging: la cabecera que se registra sola

`IHttpClientFactory` instrumenta cada cliente con dos *loggers*. Registran la URI a nivel `Information` y **las cabeceras completas a nivel `Trace`**, sin redactar nada por defecto. Un `appsettings.Development.json` con el nivel bajado que se cuela en producción vuelca tu clave al log en la primera llamada:

```
trce: System.Net.Http.HttpClient.pasarela.ClientHandler[100]
      Sending HTTP request POST https://api.pasarela-pago.com/v1/cargos
      api-key: sk_live_9f2a41c8e7b0             ← ahí está
```

La corrección no es "no bajes nunca el nivel de log" — es declarar qué cabeceras hay que enmascarar, para que dé igual quién baje el nivel:

```csharp
// ✅ Registro de la integración con las cabeceras sensibles redactadas
builder.Services.AddHttpClient("pasarela", c =>
    {
        c.BaseAddress = new Uri("https://api.pasarela-pago.com/");
        c.DefaultRequestHeaders.Add("api-key", clave);
    })
    .RedactLoggedHeaders(new[] { "api-key", "Authorization", "Set-Cookie" })
    .ConfigurePrimaryHttpMessageHandler(() =>
        new HttpClientHandler { AllowAutoRedirect = false });
```

`RedactLoggedHeaders` sustituye el valor por `*` en la salida del logger, dejando el nombre visible. Sigues viendo que la cabecera iba puesta — que es lo que necesitas para depurar — sin ver su contenido.

Y cuando registres tú la llamada, registra **el hecho**, no el objeto:

```csharp
// ❌ Serializa la petición entera, cabeceras incluidas
logger.LogInformation("Llamando a la pasarela: {@Request}", request);

// ✅ Host, operación, resultado y un identificador de correlación
logger.LogInformation("Cargo del pedido {Pedido} en {Host}: {Status}",
    "4711", request.RequestUri!.Host, (int)respuesta.StatusCode);
```

El segundo produce `Cargo del pedido 4711 en api.pasarela-pago.com: 402` — suficiente para diagnosticar, inútil para un atacante. Cómo elegir qué campos son seguros y cómo aplicar máscaras de forma sistemática lo desarrolla [Logging estructurado](../../devops/observabilidad/Logging-Estructurado.md).

## 4. Excepciones y páginas de error

Una excepción es un log que no controlas. Si has metido la clave en la URL (error 1), aparecerá sola en cuanto algo falle:

```csharp
// ❌ El mensaje de la excepción arrastra la URL completa
throw new InvalidOperationException($"Fallo al llamar a {url}");
// → "Fallo al llamar a https://api.pasarela-pago.com/v1/cargos?api_key=sk_live_9f2a41c8e7b0&pedido=4711"
```

```csharp
// ✅ Identifica la operación, no la petición literal
throw new PasarelaException($"Fallo al cobrar el pedido 4711 ({(int)respuesta.StatusCode})");
```

Dos reglas que cierran esta vía del todo:

- **Nunca devuelvas al cliente el detalle de una excepción interna.** En ASP.NET Core, `UseDeveloperExceptionPage` solo debe estar activo en `Development`; en producción, un `ProblemDetails` genérico con un identificador de traza. La página de desarrollo muestra el `HttpRequestMessage` con sus cabeceras.
- **Cuidado con los servicios de errores externos.** Un capturador de excepciones que envía el contexto completo a un SaaS replica la fuga fuera de tu infraestructura. Configura su lista de campos sensibles.

## 5. TLS: no desactives la validación de certificado

Aparece siempre igual: la pasarela tiene un certificado que la máquina de desarrollo no reconoce, alguien busca el error, encuentra esto y lo pega.

```csharp
// ❌ Desactiva la comprobación del certificado — acepta CUALQUIER servidor
var handler = new HttpClientHandler
{
    ServerCertificateCustomValidationCallback = (_, _, _, _) => true
};
```

Ese *callback* elimina la única garantía de que estás hablando con `api.pasarela-pago.com` y no con alguien interpuesto en la red. TLS deja de autenticar al servidor y solo cifra; el atacante presenta un certificado autofirmado, tu cliente lo acepta y le entrega la `api-key` en la primera petición. .NET llama al atajo equivalente `DangerousAcceptAnyServerCertificateValidator`, y ese nombre es una advertencia deliberada.

**"Es solo temporal, en desarrollo" no es una excusa válida**, por dos motivos concretos: la línea sobrevive al *merge* porque no rompe nada y nadie la nota, y la clave que se filtra en desarrollo suele ser la misma que en producción cuando no hay claves separadas por entorno.

```csharp
// ✅ Confía en un certificado concreto, no en todos
var handler = new HttpClientHandler();
handler.ServerCertificateCustomValidationCallback = (_, cert, _, errores) =>
    errores == SslPolicyErrors.None
    || cert?.Thumbprint == huellaDelCertificadoDePruebas;
```

Esto acepta el certificado de tu entorno de pruebas y **solo ese**: cualquier otro sigue rechazándose. La alternativa mejor es instalar la CA de pruebas en el almacén de confianza de la máquina y no tocar el código.

## 6. SSRF: cuando el destino lo elige el usuario

*Server-Side Request Forgery* es hacer que tu servidor llame a una URL que decide un atacante. Si esa llamada sale con tus credenciales, no solo es una fuga de red: es una entrega de la clave.

```csharp
// ❌ El destino viene de datos de usuario
var destino = pedido.UrlNotificacionProveedor;   // "https://atacante.example/recoger"
var respuesta = await clientAutenticado.PostAsJsonAsync(destino, datos);
```

Hay una variante que engaña incluso a quien cree tener el destino fijado. `new Uri(baseUri, parte)` **descarta la base** si `parte` es una URI absoluta:

```csharp
var baseUri = new Uri("https://api.pasarela-pago.com/v1/");
var destino = new Uri(baseUri, entradaDelUsuario);
// entradaDelUsuario = "cargos/4711"              → https://api.pasarela-pago.com/v1/cargos/4711
// entradaDelUsuario = "https://atacante.example" → https://atacante.example   ← la base se ignora
```

La defensa que funciona es una **lista de permitidos de hosts**, comprobada antes de adjuntar la credencial:

```csharp
// ✅ Valida esquema y host, y solo entonces autentica
private static readonly HashSet<string> HostsPermitidos =
    new(StringComparer.OrdinalIgnoreCase) { "api.pasarela-pago.com" };

if (destino.Scheme != Uri.UriSchemeHttps || !HostsPermitidos.Contains(destino.Host))
    throw new InvalidOperationException($"Destino no permitido: {destino.Host}");

peticion.Headers.Add("api-key", clave);   // la cabecera se pone al final, nunca antes
```

Tres detalles que marcan la diferencia:

- **Valida el host, no la cadena.** `url.StartsWith("https://api.pasarela-pago.com")` acepta `https://api.pasarela-pago.com.atacante.example`. Compara `Uri.Host` con igualdad exacta.
- **Vuelve a validar tras cada redirección.** Es la razón de fondo del punto 2: sin `AllowAutoRedirect = false`, la validación del destino inicial no protege del destino final.
- **Separa los clientes.** Un `HttpClient` autenticado sirve a **un** proveedor. Para descargar de una URL firmada o de un CDN, usa un cliente sin credenciales: la firma en la query string ya autoriza esa descarga, y añadir tu clave solo la expone en un dominio ajeno.

## 7. El alcance del token: pedir el mínimo posible

Todo lo anterior reduce la probabilidad de fuga. El alcance reduce el **daño** cuando la fuga ocurre igualmente, y es la única capa que sigue funcionando cuando las demás fallan.

| Dimensión | ❌ Cómo se hace por inercia | ✅ Cómo debería hacerse |
|---|---|---|
| Permisos | Una clave de administrador que puede cobrar, reembolsar y leer todos los pedidos | Una clave que solo puede crear cargos; los reembolsos, con otra clave y otro proceso |
| Entorno | La misma clave en local, staging y producción | Una clave por entorno, para poder revocar una sin parar el resto |
| Caducidad | Una clave permanente creada hace tres años | Credenciales de vida corta ([OAuth2](../autenticacion-y-autorizacion/OAuth2.md) con *client credentials*), renovadas automáticamente |
| Origen | Aceptada desde cualquier IP | Restringida a las IP de salida de tus servidores, si el proveedor lo permite |

Con un token de solo cobros, una filtración es un incidente serio. Con la clave de administrador, es la pérdida de control de la cuenta. El coste de la diferencia son cinco minutos en el panel del proveedor.

Merece la pena saber **qué tipo de credencial** manejas, porque cambia lo que puedes hacer al filtrarse: un [JWT](../autenticacion-y-autorizacion/JWT.md) es autocontenido y por lo general **no se puede revocar** antes de que caduque, mientras que un [token opaco](../autenticacion-y-autorizacion/Tokens-Opacos.md) se invalida en el servidor del proveedor al instante. La colección de [autenticación y autorización](../autenticacion-y-autorizacion/README.md) entra en el detalle.

## 8. Qué hacer cuando un secreto se filtra

Ha pasado: la clave apareció en un log, en una captura de pantalla o en un commit. El orden importa.

1. **Asume el compromiso.** No hay forma de saber quién leyó ese log. "Era un entorno interno" o "lo borré enseguida" no son mitigaciones: el borrado no deshace una lectura previa.
2. **Rota primero, investiga después.** Genera la clave nueva en el proveedor, despliégala y **revoca la antigua explícitamente**. Crear la nueva sin revocar la vieja deja las dos activas y no arregla nada.
3. **Revisa los accesos con la clave comprometida.** Casi todas las pasarelas ofrecen un registro de peticiones por credencial. Busca llamadas desde IP o en horarios que no cuadren con tu servicio, desde la fecha más antigua en que pudo filtrarse.
4. **Corrige la vía, no solo el síntoma.** Si salió por un log, añade la redacción. Si salió por la URL, muévela a la cabecera. Rotar sin cerrar la vía garantiza una segunda filtración.
5. **Si además está en el historial de git, borrar el commit no basta.** Es el mismo razonamiento del punto 1, y lo detalla [Por qué los secretos no van a Git](../gestion-de-secretos-en-desarrollo/Por-Que-Los-Secretos-No-Van-A-Git.md).

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| La `api-key` aparece en el log de acceso del proveedor | Va en la query string en lugar de en una cabecera |
| El log local muestra `api-key: sk_live_...` | Nivel `Trace` en `System.Net.Http.HttpClient.*` sin `RedactLoggedHeaders` |
| El proveedor confirma peticiones desde una IP desconocida | Credencial filtrada; falta rotación y restricción de origen |
| `401` desde la pasarela cuando la clave es correcta | La respuesta llega tras una redirección que eliminó la cabecera `Authorization` |
| La clave aparece en un mensaje de error mostrado al usuario | Excepción con la URL completa + página de desarrollo activa en producción |
| Funciona en local y falla en producción con error de certificado | El *callback* que acepta cualquier certificado está solo en la configuración local |
| Tu servidor hace peticiones a dominios que no reconoces | SSRF: la URL de destino se construye con datos de usuario |

## Buenas prácticas avanzadas

- **Registra el cliente una sola vez, en el arranque, con todo puesto.** Un `AddHttpClient("pasarela", ...)` que ya trae `BaseAddress`, `Timeout`, `AllowAutoRedirect = false` y `RedactLoggedHeaders` convierte las decisiones de esta ficha en la opción por defecto para todo el equipo. Cuando la configuración segura es la que se obtiene sin hacer nada, deja de depender de que cada persona se acuerde.
- **Prohíbe por política que una credencial viaje en la query string, aunque el proveedor lo permita.** Algunas APIs siguen aceptando `?api_key=`, y es tentador porque simplifica las pruebas con `curl`. Un test que falle si `RequestUri.Query` contiene `key`, `token` o `secret` cuesta diez líneas y caza el error antes de que llegue a producción.
- **Verifica la fuga en vivo, no por lectura de código.** Levanta un endpoint temporal (webhook.site sirve), haz que la pasarela de pruebas redirija ahí y mira qué cabeceras llegan de verdad. Es la única forma de comprobar el comportamiento real de tu combinación de `HttpClient`, *handlers* y versión de .NET, en lugar de suponerlo.
- **Trata la validación TLS y el auto-redirect como cambios que requieren revisión explícita.** Son dos líneas que no rompen ningún test, que solucionan un problema inmediato y que nadie nota en la revisión. Una regla de análisis estático que marque `ServerCertificateCustomValidationCallback` y `AllowAutoRedirect = true` en clientes autenticados evita el caso clásico del apaño temporal que llega a producción.
- **Ensaya la rotación antes de necesitarla.** Rota la clave de la pasarela en un entorno de pruebas y cronométralo. Si el ejercicio revela que la clave está *hardcodeada* en tres sitios o que rotarla exige un despliegue completo con parada, lo has descubierto un martes por la tarde y no durante un incidente. Un proveedor que permita dos claves activas a la vez hace la rotación sin corte; comprueba si el tuyo lo soporta.

## Recursos didácticos

- [OWASP Server Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) — el material de referencia sobre SSRF. Explica por qué las listas de bloqueo de IP internas no funcionan y por qué la lista de permitidos de hosts es la única defensa fiable, que es justo el patrón del punto 6.
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) — la lista concreta de qué no registrar nunca (credenciales, tokens de sesión, datos de pago) y cómo aplicar redacción de forma sistemática. Complementa el punto 3 con criterios que van más allá de las cabeceras HTTP.
- [webhook.site](https://webhook.site) — te da una URL temporal que muestra **exactamente** qué cabeceras y qué cuerpo recibe. Es la herramienta para comprobar en vivo si tu cliente reenvía la credencial tras una redirección, en lugar de fiarte de lo que crees que hace `HttpClient`.

---

*En resumen: guardar bien un secreto no sirve de nada si lo filtras al usarlo — llévalo en una cabecera y no en la URL, corta el auto-redirect, redacta los logs, valida siempre el certificado y el host de destino, y dale el mínimo alcance posible para que el día que se escape no te cueste la cuenta.*
