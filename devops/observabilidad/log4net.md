# log4net

**¿Qué es?** log4net es la librería de logging veterana de .NET, portada desde el conocidísimo log4j de Java. Envía cada log a uno o varios destinos (los llama *appenders*) según una configuración basada en XML y una jerarquía de *loggers* por nombre.

---

## ¿Por qué existe?

Durante muchos años, antes de que existieran [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md), [Serilog](Serilog.md) o [NLog](NLog.md), log4net era el estándar de facto para loguear en .NET. Trajo al mundo .NET ideas de log4j que hoy damos por sentadas: separar el "qué logueo" del "a dónde va", configurarlo por fichero externo y organizar los *loggers* en una jerarquía por nombre.

Sigue existiendo sobre todo por inercia: aparece en aplicaciones .NET con años de vida donde cambiarlo no compensa. En proyectos nuevos, hoy se elige Serilog o NLog, que nacieron pensando en logging estructurado.

> Si vienes de Java y conoces log4j, log4net es literalmente su equivalente: los mismos conceptos (appenders, layouts, niveles) con nombres casi idénticos.

---

## ¿Cuándo y para qué se usa?

Casi siempre en código heredado: una aplicación .NET Framework que ya lo tenía montado y sigue en producción. Se usa igual que las demás: escribir logs con niveles y repartirlos a ficheros, consola, base de datos o correo. Ejemplo típico: un servicio de gestión de pedidos de hace diez años que escribe un fichero rotado por tamaño mediante un `RollingFileAppender`.

---

## Lo mínimo que necesitas saber

**1. Se obtiene un logger por clase y se loguea con niveles**

```csharp
private static readonly ILog log = LogManager.GetLogger(typeof(ServicioPedidos));

log.Info("Pedido confirmado");
log.Warn("Reintentando envío de email");
log.Error("Fallo al cobrar el pedido", ex);
```

Los niveles son `DEBUG`, `INFO`, `WARN`, `ERROR` y `FATAL` (la misma escala de severidad de [Niveles de Log](Niveles-de-Log.md), con otros nombres).

**2. Los *appenders* son los destinos, configurados en XML**

```xml
<appender name="fichero" type="log4net.Appender.RollingFileAppender">
  <file value="logs/app.log" />
  <rollingStyle value="Size" />
  <layout type="log4net.Layout.PatternLayout">
    <conversionPattern value="%date %level %logger - %message%newline" />
  </layout>
</appender>
<root>
  <level value="INFO" />
  <appender-ref ref="fichero" />
</root>
```

Un *appender* equivale al *sink* de Serilog o al *target* de NLog; el *layout* define el formato de cada línea.

**3. Los *loggers* forman una jerarquía por nombre**

Un logger llamado `MiApp.Pedidos` hereda la configuración de `MiApp`, y este de la raíz (`root`). Así se puede subir el detalle de una parte concreta de la aplicación sin tocar el resto.

---

## Lo que NO hace

- **No hace logging estructurado de forma nativa** — trata el log como texto formateado por un *layout*, no como un conjunto de campos consultables; ahí está su mayor diferencia frente a Serilog y NLog.
- **No es la opción por defecto en .NET moderno** — en proyectos nuevos se integra todo sobre `Microsoft.Extensions.Logging`; log4net vive sobre todo en código heredado.
- **No almacena ni indexa** — genera y reparte; guardar y consultar es cosa del destino.

---

*En resumen: log4net es el abuelo del logging en .NET, portado de log4j — sólido y omnipresente en código heredado, pero superado por Serilog y NLog en proyectos nuevos por no nacer orientado al logging estructurado.*
