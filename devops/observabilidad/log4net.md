# log4net

## ¿Qué es?

log4net es la librería de logging veterana de .NET, portada desde log4j de Java. Reparte cada log a uno o varios destinos —los llama *appenders*— según una configuración XML y una jerarquía de *loggers* organizada por nombre.

## ¿Por qué existe?

Durante muchos años, antes de que existieran [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md), [Serilog](Serilog.md) o [NLog](NLog.md), log4net era el estándar de facto para loguear en .NET. Trajo al mundo .NET ideas de log4j que hoy damos por sentadas: separar el "qué registro" del "a dónde va", configurarlo por fichero externo y organizar los loggers en una jerarquía que hereda configuración.

Hoy sigue existiendo por inercia. Aparece en aplicaciones .NET Framework con años de vida donde cambiarlo no compensa, y su presencia suele ser buena señal de que hay más cosas antiguas alrededor. En proyectos nuevos se elige Serilog o NLog, que nacieron pensando en logging estructurado.

> Si vienes de Java, log4net es literalmente el equivalente de log4j: los mismos conceptos —appenders, layouts, niveles, jerarquía— con nombres casi idénticos.

## ¿Cuándo y para qué se usa?

Casi siempre en código heredado que ya lo tenía montado. Esta ficha está escrita desde esa realidad: lo más probable es que te lo encuentres antes que elegirlo, y lo que necesitas es entender la configuración que hay, poder tocarla sin romper nada y saber qué opciones tienes si conviene migrar.

Como referencia rápida, el mapa con las otras dos librerías:

| Concepto | log4net | NLog | Serilog |
|---|---|---|---|
| Destino | *appender* | *target* | *sink* |
| Formato de línea | *layout* | *layout* | *output template* |
| Filtrado por origen | jerarquía de loggers | `rules` | `MinimumLevel.Override` |
| Estructurado nativo | No | Sí | Sí |

---

## Escribir logs

El patrón clásico es un campo estático por clase:

```csharp
public class ServicioPedidos
{
    private static readonly ILog log = LogManager.GetLogger(typeof(ServicioPedidos));

    public void Confirmar(int pedidoId)
    {
        log.Info($"Pedido {pedidoId} confirmado");
        log.Warn($"Reintentando envío de email del pedido {pedidoId}");
        log.Error("Fallo al cobrar el pedido", ex);
    }
}
```

Los niveles son `DEBUG`, `INFO`, `WARN`, `ERROR` y `FATAL` — la misma escala de severidad de [Niveles de Log](Niveles-de-Log.md) con otros nombres. `FATAL` es el `Critical` de .NET.

Y ahí se ve la limitación de fondo: **la firma recibe una cadena ya construida**. Esa interpolación no es una mala práctica de quien escribió el código, es la única forma que ofrece la API. log4net no tiene plantillas con nombre, así que `4711` nunca llega a ser un campo consultable: es texto dentro de una frase.

Es la diferencia estructural con Serilog y NLog, y el motivo principal por el que no se elige hoy.

## Appenders y layouts

La configuración vive en XML, en `log4net.config` o dentro del `App.config`:

```xml
<log4net>
  <appender name="fichero" type="log4net.Appender.RollingFileAppender">
    <file value="logs/tienda.log" />
    <appendToFile value="true" />
    <rollingStyle value="Composite" />
    <datePattern value="yyyyMMdd" />
    <maxSizeRollBackups value="14" />
    <maximumFileSize value="10MB" />
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%date [%thread] %-5level %logger - %message%newline%exception" />
    </layout>
  </appender>

  <root>
    <level value="INFO" />
    <appender-ref ref="fichero" />
  </root>
</log4net>
```

El **appender** es el destino y el **layout** decide el formato de cada línea. El `conversionPattern` es una plantilla con códigos: `%date`, `%level`, `%logger`, `%message`, `%exception`.

Dos detalles del `RollingFileAppender` que conviene revisar en cualquier configuración heredada:

- **`rollingStyle="Composite"`** rota por fecha **y** por tamaño. Con solo `Size` un fichero muy activo rota varias veces al día y cuesta encontrar nada; con solo `Date`, un día malo genera un fichero de gigabytes.
- **`maxSizeRollBackups`** limita cuántos ficheros antiguos se conservan. Sin este límite —y falta a menudo— los logs llenan el disco, que tumba todo lo que corra en la máquina.

Y el `%exception` al final del patrón: sin él, la excepción que pasas como segundo argumento **no se escribe**. Es un fallo silencioso muy común en configuraciones heredadas: se registra el mensaje y se pierde el *stack trace*.

## La jerarquía de loggers

Este es el mecanismo propio de log4net. Los loggers se organizan por nombre con puntos, formando un árbol donde cada nodo hereda del padre:

```
root
└── MiTienda
    ├── MiTienda.Aplicacion
    │   └── MiTienda.Aplicacion.ConfirmarPedido
    └── MiTienda.Infraestructura
```

Un logger sin configuración propia usa la del ancestro más cercano. Eso permite subir el detalle de una parte sin tocar el resto:

```xml
<root>
  <level value="WARN" />
  <appender-ref ref="fichero" />
</root>

<logger name="MiTienda.Aplicacion">
  <level value="DEBUG" />
</logger>
```

Con esto, todo va a `WARN` salvo `MiTienda.Aplicacion` y sus descendientes, que van a `DEBUG`. Es la misma idea que el filtrado por categoría de M.E.L.

Y la trampa clásica: **los appenders también se heredan y se acumulan**. Si defines un `appender-ref` en un logger hijo y su padre ya tenía otro, el evento se escribe en los dos. Para cortar la herencia hay que decirlo explícitamente:

```xml
<logger name="MiTienda.Aplicacion" additivity="false">
  <level value="DEBUG" />
  <appender-ref ref="ficheroAplicacion" />
</logger>
```

`additivity="false"` es la causa —y la solución— de la mayoría de los "cada log aparece dos veces" en aplicaciones con log4net.

## Inicializarlo

log4net no lee su configuración solo: hay que decírselo, normalmente con un atributo de ensamblado en `AssemblyInfo.cs` o `Program.cs`:

```csharp
[assembly: log4net.Config.XmlConfigurator(ConfigFile = "log4net.config", Watch = true)]
```

`Watch = true` es el equivalente del `autoReload` de NLog: recarga al cambiar el fichero.

Si no se inicializa, log4net **no escribe nada y no da ningún error**. Y cuando escribe mal, su diagnóstico interno es lo único que lo explica:

```xml
<appSettings>
  <add key="log4net.Internal.Debug" value="true" />
</appSettings>
```

## Convivir con .NET moderno

Si te encuentras log4net en una aplicación que ya usa `ILogger<T>`, no hace falta elegir: existe un proveedor que engancha log4net por debajo, igual que Serilog o NLog.

```bash
dotnet add package Microsoft.Extensions.Logging.Log4Net.AspNetCore
```

```csharp
builder.Logging.ClearProviders();
builder.Logging.AddLog4Net("log4net.config");
```

Con eso, el código nuevo inyecta `ILogger<T>` con normalidad y los logs acaban en los appenders que ya estaban configurados. Es la estrategia sensata para modernizar sin migración masiva: **el código nuevo usa la abstracción estándar y la configuración antigua sigue funcionando**. El día que quieras cambiar a Serilog, tocas solo esa línea del arranque.

## Migrar a Serilog o NLog

Si el proyecto va a seguir vivo, migrar suele compensar, sobre todo por el logging estructurado. El camino con menos riesgo tiene tres pasos:

1. **Introducir `ILogger<T>` en el código nuevo**, con log4net todavía por debajo mediante el proveedor de arriba. Nada se rompe y deja de crecer la deuda.
2. **Sustituir el proveedor** por Serilog o NLog en el arranque. El código que ya usa `ILogger<T>` sigue funcionando sin tocarse; solo hay que traducir la configuración XML a la nueva.
3. **Reescribir las llamadas antiguas** (`log.Info($"...")`) a plantillas con nombre, poco a poco. Este paso es el único que da valor real —convierte los logs en datos consultables— y también el único que se puede hacer por partes, sin prisa.

El error a evitar es intentar los tres pasos a la vez en una rama larga.

## Buenas prácticas avanzadas

- **Revisa que el patrón incluya `%exception`.** Sin él, `log.Error("mensaje", ex)` escribe la frase y descarta el *stack trace* completo, que es justo lo que hace falta para arreglar el fallo. Es un olvido silencioso que puede llevar años en producción sin que nadie lo note, hasta la primera incidencia seria en que la traza habría ahorrado medio día.
- **Comprueba `additivity` cuando veas logs duplicados.** Los appenders se heredan y se suman por el árbol de loggers, así que un `appender-ref` en un hijo cuyo padre ya tenía otro escribe el evento dos veces. Duplica el coste de almacenamiento y, peor, hace creer que un problema ocurrió el doble de veces de las reales al contar líneas.
- **Limita el número y tamaño de los ficheros de rotación.** `maxSizeRollBackups` y `maximumFileSize` no vienen acotados de forma útil por defecto, y un bucle de errores genera gigabytes en minutos. Un disco lleno no solo detiene el logging: tumba la base de datos y todo lo demás que comparta la máquina, y el diagnóstico apunta a cualquier sitio menos a los logs.
- **No uses `%stacktrace` ni `%location` en el patrón general.** Obligan a inspeccionar la pila de llamadas en **cada** evento, lo que en código frecuente se nota mucho. La información de dónde saltó una excepción ya viene en `%exception`; el resto de las veces no compensa el coste.
- **Envuélvelo tras `ILogger<T>` antes de plantearte migrarlo.** Añadir el proveedor de log4net a M.E.L cuesta dos líneas y desacopla inmediatamente el código nuevo de la librería. Es lo que convierte una migración de "reescribir cada llamada del proyecto" en "cambiar una línea del arranque cuando toque", y permite empezar a escribir logs estructurados hoy sin haber terminado de quitar log4net.

## Recursos didácticos

- [Manual de configuración de log4net](https://logging.apache.org/log4net/release/manual/configuration.html) — la referencia oficial de appenders, layouts y jerarquía; densa pero completa, y la única fuente fiable para las opciones del `RollingFileAppender`.
- [Referencia de `PatternLayout`](https://logging.apache.org/log4net/release/sdk/html/T_log4net_Layout_PatternLayout.htm) — la tabla de todos los códigos `%` disponibles, imprescindible para descifrar el `conversionPattern` que te encuentres en un proyecto heredado.
- [Seq](https://datalust.co/seq) — comparar lo que llega desde log4net con lo que llega desde Serilog en el mismo agregador enseña de golpe qué se gana con el logging estructurado, y suele ser el argumento que convence para migrar.

---

*En resumen: log4net es el abuelo del logging en .NET, sólido y omnipresente en código heredado — trátalo tras `ILogger<T>` para desacoplarte de él, y revisa que el patrón lleve `%exception` y que la rotación tenga límites.*
