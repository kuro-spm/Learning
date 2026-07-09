# Niveles de Log

## ¿Qué es?

Un nivel de log es una etiqueta de severidad que se asigna a cada mensaje (`Debug`, `Information`, `Warning`, `Error`...) para indicar cuánto importa. Sirve para dos cosas: decir de un vistazo la gravedad de lo que ha pasado y filtrar en cada entorno qué mensajes se guardan y cuáles se descartan.

## ¿Por qué existe?

Sin niveles, todos los logs valdrían lo mismo: el detalle de depuración que solo interesa en local acabaría mezclado con el error que despierta a alguien de madrugada. En producción ese ruido es un problema real, porque el volumen es enorme y guardar cada línea cuesta dinero y esconde las señales importantes.

Los niveles resuelven esto poniendo un dial: escribes en el código *toda* la información que puede ser útil, cada línea con su severidad, y luego decides por configuración —sin tocar el código— a partir de qué nivel se guarda en cada entorno. En local quieres verlo todo; en producción, solo de `Warning` para arriba.

> Si conoces la configuración de compilación `Debug` vs `Release` de .NET, la idea es parecida: el mismo código fuente se comporta distinto según un ajuste externo, no según una rama `if` que escribes a mano.

## ¿Cuándo y para qué se usa?

En cualquier aplicación que genere logs. Se usa al escribir cada mensaje (eliges su nivel) y al configurar cada entorno (eliges el nivel mínimo que se guarda). Ejemplos: en una tienda online, el detalle de cada consulta SQL va en `Debug` (útil depurando, invisible en producción); un pago rechazado por fondos insuficientes es `Warning`; que la pasarela de pago no responda es `Error`; que la base de datos entera esté caída es `Critical`.

## Lo mínimo que necesitas saber

**1. Los niveles habituales, de menos a más grave**

En .NET (`Microsoft.Extensions.Logging`) el orden es:

```csharp
logger.LogTrace("Entrando en el método con {ProductId}", productId);   // detalle extremo, casi nunca en producción
logger.LogDebug("Consulta SQL: {Query}", query);                       // diagnóstico para desarrollo
logger.LogInformation("Pedido {OrderId} confirmado", orderId);         // eventos de negocio normales
logger.LogWarning("Reintento {Intento} de envío de email", intento);   // algo raro, pero se ha podido continuar
logger.LogError(ex, "Fallo al cobrar el pedido {OrderId}", orderId);   // una operación ha fallado
logger.LogCritical(ex, "Base de datos inaccesible");                   // el sistema no puede seguir funcionando
```

**2. El nivel mínimo se configura por entorno, sin tocar el código**

```json
// appsettings.Development.json  → se ve todo desde Debug
{ "Logging": { "LogLevel": { "Default": "Debug" } } }

// appsettings.Production.json   → solo Warning y por encima
{ "Logging": { "LogLevel": { "Default": "Warning" } } }
```

Con `Default: "Warning"`, las llamadas a `LogTrace`, `LogDebug` y `LogInformation` no se escriben en producción, aunque el código las siga ejecutando.

**3. La regla es "este nivel y todos los más graves"**

Un mínimo de `Warning` guarda `Warning`, `Error` y `Critical`, y descarta `Trace`, `Debug` e `Information`. No se eligen niveles sueltos: se elige un umbral.

**4. `Error` vs `Critical`: ¿ha fallado una operación o el sistema entero?**

`Error` es una operación concreta que ha fallado pero el servicio sigue en pie (un pedido no se ha podido cobrar). `Critical` es que la aplicación no puede seguir funcionando (se ha quedado sin conexión a la base de datos, sin memoria...). La diferencia importa porque `Critical` suele ir enganchado a una alerta que avisa a alguien.

**5. Los nombres cambian entre librerías, el concepto no**

`Debug`, `Information` y `Error` de .NET se llaman `DEBUG`, `INFO` y `ERROR` en log4net, y `Verbose`/`Debug`/`Information`/`Fatal` en Serilog. Son etiquetas distintas para la misma escala de severidad.

## Lo que NO hace

- **No decide por ti qué nivel merece cada log** — que existan los niveles no impide poner un `LogInformation` dentro de un bucle que se ejecuta un millón de veces; elegir bien la severidad sigue siendo tuyo.
- **No cambia el contenido del mensaje** — un log mal escrito (sin contexto, sin campos) no mejora por subirlo a `Error`; el nivel es severidad, no calidad.
- **No es una jerarquía universal de nombres** — no des por hecho que `Warning` existe con ese nombre exacto en todas las plataformas.

## Buenas prácticas avanzadas

- **Reserva `Information` para eventos de negocio, no para trazar el flujo del código** — "pedido confirmado", "usuario registrado" son `Information`; "entrando en el método X", "valor de la variable Y" son `Debug` o `Trace`. Es el error más común: llenar producción de `Information` de diagnóstico convierte el nivel en ruido y obliga a subir el umbral, con lo que se pierden también los eventos que sí importaban.
- **Ajusta el nivel mínimo por categoría, no solo el global** — las librerías de terceros (Entity Framework registrando cada SQL, ASP.NET Core registrando cada petición) generan muchísimo `Information`; sube *su* umbral a `Warning` dejando el de tu propio código en `Information`, en vez de subir el global y quedarte a ciegas sobre tu aplicación.
- **`LogError` casi siempre debe recibir la excepción como primer argumento** — `logger.LogError(ex, "…")` guarda el *stack trace* completo; `logger.LogError("Ha fallado: {Mensaje}", ex.Message)` se queda solo con el texto y pierde dónde saltó el fallo, que suele ser justo lo que necesitas para arreglarlo.
- **No uses el nivel como si fuera un canal de métricas** — contar cuántos `Warning` hubo para deducir "cuántos pagos se reintentaron" es frágil; si necesitas medir la frecuencia de algo, eso es una métrica (ver [Métricas](Metricas.md)), no un recuento de líneas de log.

## Recursos didácticos

- La sección [Log Level](https://learn.microsoft.com/dotnet/core/extensions/logging#log-level) de la documentación de Microsoft, con la tabla de los seis niveles y cuándo usar cada uno.

---

*En resumen: el nivel de log es el dial de severidad de cada mensaje — escribes todo en el código y decides por configuración, entorno a entorno, a partir de qué gravedad se guarda.*
