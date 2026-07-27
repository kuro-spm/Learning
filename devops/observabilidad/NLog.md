# NLog

## ¿Qué es?

NLog es una librería de logging para .NET, alternativa a [Serilog](Serilog.md), con mucho recorrido en el ecosistema. Reparte cada log a distintos destinos —los llama *targets*— según reglas que viven tradicionalmente en un fichero XML externo, editable sin recompilar.

## ¿Por qué existe?

El proveedor de serie de [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md) escribe a consola y poco más. Para producción hace falta enviar los logs a ficheros rotados, a una base de datos o a un sistema centralizado, y poder cambiar ese enrutado sin desplegar.

NLog cubre ese terreno igual que Serilog, y con un rasgo que lo distingue: **su configuración vive fuera del código, en un `nlog.config`, y puede recargarse en caliente**. Un equipo de operaciones puede subir el detalle de una parte concreta durante una incidencia editando un fichero, sin pasar por un despliegue ni reiniciar el proceso.

> Si conoces Serilog, NLog cubre lo mismo. La diferencia práctica: Serilog empuja a configurarse en código y NLog en XML externo. Ambos admiten las dos formas, pero cada uno va cómodo en la suya.

## ¿Cuándo y para qué se usa?

En aplicaciones .NET donde se valora reconfigurar el logging por fichero sin tocar el binario, y en proyectos con años de vida donde ya estaba antes de que Serilog se popularizara.

La elección entre uno y otro rara vez es técnica: los dos hacen logging estructurado, los dos tienen un catálogo amplio de destinos y los dos se integran igual con `ILogger<T>`. Pesa más si el equipo de operaciones va a tocar la configuración (favorece NLog) o si el equipo prefiere tenerlo todo en código y revisado por control de versiones (favorece Serilog).

---

## Montarlo en ASP.NET Core

```bash
dotnet add package NLog.Web.AspNetCore
```

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Logging.ClearProviders();
builder.Host.UseNLog();

var app = builder.Build();
```

`ClearProviders()` evita que los logs se escriban además por los proveedores por defecto, duplicando cada línea.

Y en `Program.cs` conviene envolverlo igual que con Serilog, para no perder los fallos de arranque:

```csharp
var logger = NLog.LogManager.Setup().LoadConfigurationFromAppSettings().GetCurrentClassLogger();

try
{
    logger.Info("Arrancando la aplicación");
    // ... construir y ejecutar la app
}
catch (Exception ex)
{
    logger.Error(ex, "La aplicación no ha podido arrancar");
    throw;
}
finally
{
    NLog.LogManager.Shutdown();   // vacía los búferes
}
```

`LogManager.Shutdown()` cumple el mismo papel que `Log.CloseAndFlush()` en Serilog: sin él, los *targets* asíncronos pueden dejar sin escribir justo los últimos eventos.

Hecho esto, el código de negocio no cambia: sigue inyectando `ILogger<T>`.

## `nlog.config`: targets y rules

El fichero tiene dos secciones, y entender la relación entre ellas es entender NLog.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      autoReload="true"
      internalLogLevel="Warn"
      internalLogFile="logs/nlog-internal.log">

  <targets>
    <target name="fichero" xsi:type="File"
            fileName="logs/tienda-${shortdate}.log"
            maxArchiveFiles="14"
            archiveEvery="Day" />

    <target name="consola" xsi:type="Console" />
  </targets>

  <rules>
    <logger name="Microsoft.*" maxlevel="Info" final="true" />
    <logger name="*" minlevel="Info" writeTo="fichero" />
    <logger name="*" minlevel="Warn" writeTo="consola" />
  </rules>

</nlog>
```

**Los `targets` son los destinos.** Equivalen a los *sinks* de Serilog: fichero, consola, base de datos, red, correo. El `${shortdate}` es un *layout renderer*, una plantilla que NLog resuelve en tiempo de ejecución.

**Las `rules` deciden qué evento va a qué target.** Se evalúan **en orden**, de arriba abajo, y ahí está el detalle que hay que entender:

```xml
<logger name="Microsoft.*" maxlevel="Info" final="true" />
```

Esa primera regla no tiene `writeTo`: descarta. Dice "todo lo de categoría `Microsoft.*` hasta nivel `Info`, tíralo y **no sigas evaluando reglas** (`final="true"`)". Es la forma de NLog de filtrar el ruido de las librerías de terceros, equivalente al `MinimumLevel.Override` de Serilog. Sin ella, el fichero se llena del registro de cada petición y cada consulta de EF Core.

Las dos reglas siguientes mandan a destinos distintos con umbrales distintos: todo lo `Info` o superior al fichero, y solo `Warn` o superior también a consola.

Y los dos atributos del elemento raíz que conviene poner siempre:

- **`autoReload="true"`** — NLog vigila el fichero y aplica los cambios sin reiniciar.
- **`internalLogFile`** — cuando NLog no escribe y no dice por qué (un *target* mal configurado, una ruta sin permisos), su log interno es lo único que lo explica. Es el equivalente al `SelfLog` de Serilog y ahorra tardes enteras.

## Logging estructurado

NLog también captura propiedades con nombre, con la misma sintaxis:

```csharp
logger.LogInformation("Pedido {PedidoId} confirmado por {Total}", pedido.Id, pedido.Total);
```

Pero hay un paso extra que no está en Serilog: **por defecto, el layout de fichero solo escribe el mensaje ya formateado**. Para que las propiedades salgan como campos hay que decirlo en el *layout*:

```xml
<target name="fichero" xsi:type="File"
        fileName="logs/tienda-${shortdate}.log">
  <layout xsi:type="JsonLayout" includeEventProperties="true">
    <attribute name="time" layout="${longdate}" />
    <attribute name="level" layout="${level:upperCase=true}" />
    <attribute name="logger" layout="${logger}" />
    <attribute name="message" layout="${message}" />
  </layout>
</target>
```

`includeEventProperties="true"` es la clave: sin él, `PedidoId` se queda dentro del texto y pierdes todo lo que hace útil el [logging estructurado](Logging-Estructurado.md). Es un olvido frecuente al migrar de un layout de texto.

## Contexto por ámbito

El equivalente de `LogContext` de Serilog es el `ScopeContext`:

```csharp
using (NLog.ScopeContext.PushProperty("PedidoId", pedido.Id))
{
    logger.LogInformation("Stock reservado");
    logger.LogInformation("Pago cobrado");
}
```

Y como NLog está enganchado a M.E.L, el `BeginScope` estándar también funciona, que es lo preferible para no atar el código de negocio a una librería concreta:

```csharp
using (logger.BeginScope(new Dictionary<string, object> { ["PedidoId"] = pedido.Id }))
{
    logger.LogInformation("Stock reservado");
}
```

Para que esas propiedades lleguen al fichero hace falta `includeScopeProperties="true"` en el `JsonLayout`.

## Rendimiento: `AsyncWrapper`

Igual que en Serilog, escribir a disco o a red de forma síncrona añade latencia a la petición. NLog lo resuelve envolviendo el *target*:

```xml
<targets>
  <target name="ficheroAsync" xsi:type="AsyncWrapper" queueLimit="10000" overflowAction="Discard">
    <target xsi:type="File" fileName="logs/tienda-${shortdate}.log" />
  </target>
</targets>
```

Los dos parámetros del envoltorio son una decisión de diseño real: `queueLimit` acota la memoria que puede ocupar la cola, y `overflowAction` decide qué hacer al llenarse. `Discard` tira eventos (protege la aplicación), `Block` frena a quien loguea (protege los logs). En un servicio que atiende peticiones, `Discard` casi siempre es la respuesta correcta: perder logs es mejor que degradar el servicio.

Existe el atajo `<targets async="true">`, que envuelve todos los targets de la sección de golpe.

## Cuidado con los *layout renderers* caros

NLog trae muchísimos, y algunos obligan a inspeccionar la pila de llamadas en cada evento:

```xml
<!-- ⚠️ Caros: fuerzan a leer el stack trace en cada log -->
<attribute name="metodo" layout="${callsite}" />
<attribute name="linea" layout="${callsite-linenumber}" />
<attribute name="pila" layout="${stacktrace}" />
```

En código que se ejecuta miles de veces por segundo, esto se ve en el perfil. Y suele ser innecesario: para una excepción, `${exception:format=ToString}` ya incluye el *stack trace* completo. Reserva los de *callsite* para los niveles altos o para depuración puntual.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| No se escribe nada | El `nlog.config` no se copia a la salida: falta `CopyToOutputDirectory` |
| Las propiedades no salen como campos | Falta `includeEventProperties="true"` en el `JsonLayout` |
| Los logs salen duplicados | Falta `ClearProviders()`, o dos reglas mandan al mismo target |
| El fichero se llena de ruido de Microsoft | Falta la regla de descarte con `final="true"` |
| Se pierden los últimos logs al cerrar | Falta `LogManager.Shutdown()` |
| Cambio el config y no pasa nada | Falta `autoReload="true"` |

El primero es el que más desconcierta porque no da ningún error. El `nlog.config` tiene que acabar junto al ejecutable:

```xml
<ItemGroup>
  <None Update="nlog.config" CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

## Buenas prácticas avanzadas

- **Activa el log interno con un nivel razonable y déjalo puesto.** `internalLogLevel="Warn"` con `internalLogFile` cuesta dos atributos y es lo único que explica por qué NLog no escribe: una ruta sin permisos, un target con un nombre mal escrito, un fallo de conexión con la base de datos de destino. Sin él, NLog falla en silencio por diseño, y diagnosticarlo a ciegas puede llevar horas.
- **Usa `final="true"` para descartar el ruido antes de que llegue a los targets.** Filtrar en la regla es mucho más barato que dejar que el evento se formatee y lo descarte el destino. La regla de `Microsoft.*` al principio del bloque de `rules` es prácticamente obligatoria en cualquier aplicación ASP.NET Core, y su ausencia es la causa de los ficheros de log que crecen sin explicación.
- **`autoReload` es potente y hay que devolverlo a su sitio.** Poder subir a `Debug` una categoría en producción durante una incidencia es exactamente para lo que sirve. El problema es dejarlo puesto: el volumen y la factura crecen y nadie recuerda haberlo activado tres meses después. Anota el cambio junto al incidente y revierte al cerrarlo.
- **Define `queueLimit` y `overflowAction` explícitamente en los targets asíncronos.** Los valores por defecto pueden hacer que un pico de logs consuma memoria o que la aplicación se frene esperando al disco, y cuál de las dos cosas prefieres es una decisión que deberías tomar tú y no descubrir en producción. Para un servicio que atiende peticiones, `Discard` con un límite generoso es lo sensato.
- **No mezcles la API nativa de NLog con `ILogger<T>` en el mismo código.** `LogManager.GetCurrentClassLogger()` funciona y ata ese código a NLog para siempre, además de saltarse el filtrado por categoría de M.E.L. Usar `ILogger<T>` en toda la aplicación mantiene la opción de cambiar de librería con una línea en el arranque, que es justo lo que da valor a la abstracción.

## Recursos didácticos

- [Wiki de NLog](https://github.com/NLog/NLog/wiki) — el catálogo completo de *targets* y *layout renderers* con ejemplos de configuración. Es una de las mejores wikis de librería del ecosistema .NET.
- [Referencia de layout renderers](https://nlog-project.org/config/?tab=layout-renderers) — buscador de los `${...}` disponibles, útil para no reinventar algo que ya existe (y para ver cuáles marcan que son caros).
- [Seq](https://datalust.co/seq) — con `NLog.Targets.Seq` se ve inmediatamente si el `JsonLayout` está exportando las propiedades como campos o solo como texto, que es el fallo más común al configurar NLog para logging estructurado.

---

*En resumen: NLog hace lo mismo que Serilog con la configuración en un XML externo recargable en caliente — activa su log interno, filtra el ruido con `final="true"` y no olvides `includeEventProperties` si quieres estructura.*
