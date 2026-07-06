# OpenTelemetry

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

OpenTelemetry (a menudo abreviado como **OTel**) es un estándar abierto y vendor-neutral para instrumentar aplicaciones y generar logs, métricas y trazas de forma unificada, con un formato común independiente del backend al que luego se envíen los datos.

## ¿Por qué existe?

Antes de OpenTelemetry, cada proveedor de observabilidad (Datadog, New Relic, Application Insights, Jaeger...) tenía su propio SDK de instrumentación. Instrumentar una aplicación para un proveedor y luego querer cambiar a otro implicaba reescribir buena parte del código de instrumentación, porque cada uno tenía su propia forma de crear spans, contar métricas o etiquetar logs.

OpenTelemetry nace de la fusión de dos proyectos anteriores (OpenTracing y OpenCensus) con un objetivo claro: que instrumentar el código sea independiente de dónde acaben los datos. Instrumentas una vez con la API de OpenTelemetry, y decides el destino (Jaeger, Prometheus, Application Insights, un backend comercial) solo en la configuración, sin tocar el código de negocio.

> Es al mundo de la observabilidad lo que un *driver* estándar es a una impresora: el programa que imprime no necesita saber la marca exacta de la impresora, solo habla el protocolo común; quien cambia la impresora, no toca el programa.

## ¿Cuándo y para qué se usa?

Cuando quieres instrumentar una aplicación de forma que no quede atada a un proveedor concreto de observabilidad, o cuando ese proveedor puede cambiar con el tiempo (empezar con una herramienta gratuita y migrar más adelante a una de pago, o centralizar la telemetría de varios equipos que usan backends distintos). Es, hoy, la forma estándar de instrumentar servicios .NET, Java, Node o Go que necesitan logs, métricas y trazas correlacionados entre sí.

## Lo mínimo que necesitas saber

**1. Instrumentación de la aplicación (el SDK)**

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation());
```

Con esto, las peticiones HTTP entrantes y salientes generan spans y métricas automáticamente, sin escribir código manual para cada una.

**2. Los "exporters" son el destino de los datos**

Igual que los *sinks* de Serilog, un *exporter* decide a dónde van logs, métricas o trazas: consola (para depurar en local), un colector propio, o directamente un backend como Jaeger o Application Insights.

```csharp
.WithTracing(tracing => tracing
    .AddOtlpExporter(options => options.Endpoint = new Uri("http://localhost:4317")));
```

**3. El OpenTelemetry Collector: una pieza intermedia opcional**

El *Collector* es un proceso independiente que recibe la telemetría de las aplicaciones, la procesa (filtra, agrupa, transforma) y la reenvía a uno o varios backends. Permite cambiar de backend sin tocar ni reiniciar las aplicaciones.

```
[App 1] ─┐
[App 2] ─┼─> [OpenTelemetry Collector] ──> [Backend elegido: Jaeger, Prometheus, ...]
[App 3] ─┘
```

**4. Instrumentación manual con la API común**

```csharp
using var activity = MyActivitySource.StartActivity("pagos.Cobrar");
activity?.SetTag("orderId", pedido.Id);
```

`Activity` es la abstracción de .NET que se corresponde con un *span* de OpenTelemetry; el propio `System.Diagnostics` del runtime ya es compatible con el estándar.

**5. Un mismo estándar, tres señales**

La misma configuración e infraestructura sirve para logs, métricas y trazas, lo que facilita que compartan el mismo contexto de correlación (`traceId`) entre las tres señales.

## Lo que NO hace

- **No almacena ni visualiza nada por sí mismo** — es el estándar de instrumentación y transporte; necesitas un backend (Jaeger, Prometheus, Grafana, un producto comercial) para guardar y consultar los datos.
- **No sustituye a Serilog ni a otras librerías de logging** — la parte de logs de OpenTelemetry suele integrarse *con* el sistema de logging existente, no reemplazarlo de la noche a la mañana.
- **No garantiza compatibilidad total entre todos los lenguajes** — el nivel de madurez de logs, métricas y trazas varía según el lenguaje; en .NET, tracing y métricas están más maduros que la parte de logs.
- **No configura el Collector por ti** — si decides usarlo, es una pieza de infraestructura más que hay que desplegar y mantener.

## Buenas prácticas avanzadas

- **Empieza sin Collector y añádelo cuando de verdad lo necesites** — exportar directamente desde la aplicación al backend es más simple y válido para empezar; el Collector aporta valor cuando necesitas procesar la telemetría de forma centralizada (filtrar, enriquecer, redirigir a varios backends) sin tocar cada aplicación.
- **Usa `Resource` para identificar de dónde viene cada dato** — sin marcar cada servicio con su nombre, versión y entorno (`service.name`, `service.version`, `deployment.environment`), toda la telemetría de tu sistema distribuido se mezcla sin forma de saber qué instancia o versión la generó.
- **Aprovecha que `Activity` es nativo de .NET antes de instrumentar a mano** — buena parte de las librerías del ecosistema (`HttpClient`, ASP.NET Core, Entity Framework) ya crean `Activity` por debajo; activar su instrumentación de OpenTelemetry suele dar trazas útiles antes de escribir una sola línea de instrumentación manual.
- **Trata el nombre de tus métricas y atributos como un contrato** — cambiar el nombre de una métrica o de una etiqueta rompe silenciosamente los dashboards y alertas que dependían de ella; sigue las convenciones de semántica de OpenTelemetry (`http.method`, `http.status_code`) en vez de inventar nombres propios cuando ya existe un estándar para ese dato.

---

*En resumen: OpenTelemetry es el estándar común que permite instrumentar logs, métricas y trazas una sola vez y decidir después, solo con configuración, a qué backend de observabilidad van a parar.*
