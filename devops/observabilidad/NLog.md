# NLog

## ¿Qué es?

NLog es una librería de logging para .NET, alternativa a [Serilog](Serilog.md), con mucho recorrido en el ecosistema. Captura cada log y lo reparte a distintos destinos (los llama *targets*) según reglas que se pueden definir en un fichero de configuración XML, sin recompilar la aplicación.

## ¿Por qué existe?

El proveedor de logging de serie de [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md) es deliberadamente básico: escribe a consola o al depurador, poco más. Para producción hace falta enviar los logs a ficheros rotados, a una base de datos o a un sistema centralizado, y poder cambiar ese enrutado sin tocar el código.

NLog resuelve eso, y con un rasgo que lo distingue: su configuración vive tradicionalmente en un `nlog.config` (XML) separado del código. Puedes cambiar a dónde van los logs, o subir el detalle de una parte concreta, editando un fichero y reiniciando, sin desplegar una versión nueva de la aplicación.

> Si conoces Serilog, NLog cubre el mismo terreno: la diferencia práctica es que Serilog empuja a configurarse en código y NLog empuja a configurarse en XML externo, aunque ambos admiten los dos estilos.

## ¿Cuándo y para qué se usa?

En aplicaciones .NET que necesitan logging de producción y donde se valora poder reconfigurar los destinos por fichero (equipos de operaciones que ajustan el logging sin pasar por un despliegue). Es habitual encontrarlo en proyectos con años de vida, donde ya estaba antes de que Serilog se popularizara. Ejemplos: una API de una tienda online que escribe un fichero de log rotado por día y, además, manda los errores a una tabla de base de datos.

## Lo mínimo que necesitas saber

**1. Los *targets* son los destinos**

Un *target* es a dónde va el log: fichero, consola, base de datos, red... Equivalen a los *sinks* de Serilog.

```xml
<!-- nlog.config -->
<targets>
  <target name="fichero" xsi:type="File"
          fileName="logs/app-${shortdate}.log" />
  <target name="consola" xsi:type="Console" />
</targets>
```

**2. Las *rules* deciden qué va a cada *target***

```xml
<rules>
  <!-- todo lo de nivel Info o superior va al fichero -->
  <logger name="*" minlevel="Info" writeTo="fichero" />
  <!-- solo Warning y por encima, también a consola -->
  <logger name="*" minlevel="Warn" writeTo="consola" />
</rules>
```

El `name` filtra por categoría (con comodines como `MiApp.*`), igual que en M.E.L.

**3. Se integra con `ILogger` de ASP.NET Core**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Logging.ClearProviders();
builder.Host.UseNLog();
```

Hecho esto, el resto del código sigue inyectando `ILogger<T>` con normalidad; NLog actúa como el proveedor por debajo, igual que Serilog.

**4. También hace logging estructurado con plantillas con nombre**

```csharp
logger.LogInformation("Pedido {OrderId} confirmado por {Total:C}", orderId, total);
// OrderId y Total se capturan como campos propios, no solo como texto
```

**5. La configuración puede recargarse en caliente**

Con `autoReload="true"` en el `nlog.config`, NLog detecta los cambios del fichero y los aplica sin reiniciar el proceso, útil para subir el detalle de logs mientras se investiga una incidencia.

## Lo que NO hace

- **No almacena ni indexa los logs** — como Serilog, NLog los genera y los envía; guardarlos y hacerlos consultables es cosa del destino (fichero, base de datos, Seq...).
- **No gestiona métricas ni trazas** — es una librería de logging; los otros dos pilares se cubren con [OpenTelemetry](OpenTelemetry.md).
- **No obliga a usar XML** — la configuración por fichero es su seña de identidad, pero también admite configurarse en código si se prefiere.
- **No decide qué loguear** — la elección de eventos y niveles sigue siendo de quien escribe el código.

## Buenas prácticas avanzadas

- **Usa `AsyncWrapper` para los *targets* de disco o red en producción** — escribir a fichero o a base de datos de forma síncrona en cada log añade latencia a la petición que lo generó; envolver el *target* en `<target xsi:type="AsyncWrapper">` encola el evento y lo escribe en segundo plano, para que un pico de logs no ralentice las peticiones reales.
- **Configura `minlevel` distinto por categoría, no uno global** — el ruido de las librerías de terceros ahoga los logs propios; una regla `<logger name="Microsoft.*" maxlevel="Info" final="true" />` los descarta antes de que lleguen a tus *targets*, dejando limpio el fichero de la aplicación.
- **Cuidado con `${stacktrace}` y capturas de contexto caras en rutas calientes** — algunos *layout renderers* de NLog (stack trace, información del llamador) obligan a inspeccionar la pila en cada log; en código muy frecuente eso pesa, así que resérvalos para los niveles altos donde de verdad aportan.
- **Aprovecha `autoReload` con cabeza** — poder subir el detalle en caliente es potente para diagnosticar, pero dejar el fichero en modo verboso "por si acaso" en producción genera un volumen y un coste que luego nadie recuerda haber activado; vuelve a bajarlo al cerrar la incidencia.

## Recursos didácticos

- La [wiki oficial de NLog en GitHub](https://github.com/NLog/NLog/wiki), con el catálogo completo de *targets* y *layout renderers* y ejemplos de `nlog.config`.

---

*En resumen: NLog es una librería de logging veterana de .NET equivalente a Serilog, con la particularidad de configurar sus destinos en un XML externo que puede recargarse sin desplegar la aplicación.*
