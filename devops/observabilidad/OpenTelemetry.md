# OpenTelemetry

## ¿Qué es?

OpenTelemetry (casi siempre abreviado **OTel**) es un estándar abierto y neutral respecto al proveedor para generar y enviar las tres señales de [observabilidad](Observabilidad.md) —logs, métricas y trazas— con un formato y un protocolo comunes. Instrumentas el código una vez y decides después, solo con configuración, a qué backend van los datos.

## ¿Por qué existe?

Antes de OTel, cada producto de observabilidad traía su propio SDK: Datadog el suyo, New Relic el suyo, Application Insights el suyo, Jaeger el suyo. Instrumentar la aplicación significaba llenarla de llamadas a la librería de ese proveedor concreto, repartidas por el código de negocio. Cambiar de herramienta —porque sube de precio, porque la empresa unifica, porque el equipo nuevo usa otra— implicaba reescribir toda esa instrumentación.

OpenTelemetry nace de fusionar dos proyectos previos (OpenTracing y OpenCensus) con un objetivo concreto: **separar generar la telemetría de decidir dónde acaba**. El código de negocio habla con una API estándar; el destino es una línea de configuración.

> Es lo que un *driver* estándar de impresora es al programa que imprime: el programa no sabe la marca ni el modelo, solo habla el protocolo común. Cuando cambias la impresora, no tocas el programa.

## ¿Cuándo y para qué se usa?

En cualquier servicio .NET, Java, Node o Go que vaya a producción. Hoy es la forma por defecto de instrumentar, por dos razones prácticas: **no quedarte atado** (empiezas con Jaeger y Prometheus en un contenedor, gratis, y migras luego a un producto de pago sin tocar código de negocio) y **unificar equipos** (varios servicios emitiendo el mismo formato y correlacionados por el mismo `traceId`).

Lo que OTel **no** es: un sitio donde guardar ni consultar nada. No tiene interfaz, ni base de datos, ni dibuja gráficas. Genera y transporta; guardar y visualizar es trabajo de un backend (Jaeger, Prometheus, Grafana, Application Insights, el que sea).

---

## Las piezas y cómo encajan

Esta es la parte que más confunde al empezar, porque son seis nombres para lo que uno espera que sea una sola librería. Merece la pena ordenarlos antes de escribir código.

| Pieza | Qué es | En .NET |
|---|---|---|
| **API** | Los tipos con los que tu código genera telemetría | Está **en el runtime**: `ActivitySource`, `Meter`, `ILogger` |
| **SDK** | Lo que recoge lo que emite la API, lo procesa y lo manda | Paquetes `OpenTelemetry.*`, se activa con `AddOpenTelemetry()` |
| **Instrumentación** | Librerías que emiten telemetría por ti desde ASP.NET Core, `HttpClient`, EF Core… | `OpenTelemetry.Instrumentation.*` |
| **Exporter** | El componente que envía los datos a un destino concreto | `AddOtlpExporter()`, `AddConsoleExporter()`… |
| **OTLP** | El **protocolo** estándar de OTel para transportar telemetría | gRPC en el `:4317`, HTTP en el `:4318` |
| **Collector** | Proceso aparte que recibe telemetría, la procesa y la reenvía | Un contenedor, **opcional** |

La separación API/SDK tiene una consecuencia muy práctica en .NET: **tu código de negocio no referencia OpenTelemetry**. Usa `ActivitySource` y `Meter`, que vienen en `System.Diagnostics` desde .NET 5. Si mañana quitas el SDK, el código sigue compilando y funcionando; simplemente nadie recoge lo que emite.

```
 Tu código ──► API (System.Diagnostics) ◄── Instrumentación automática
                        │                   (ASP.NET Core, HttpClient, EF Core)
                        ▼
          [ SDK de OTel ]  Resource, sampling, batching, exporter
                        │  OTLP  (:4317 gRPC / :4318 HTTP)
                        ▼
             [ Collector ]  ← opcional
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
           Jaeger            Prometheus
```

## Montarlo en .NET

El ejemplo conductor: una tienda online con los servicios `pedidos`, `pagos`, `inventario` y `emails`. Empezamos por `pedidos`.

Los paquetes:

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.Runtime
```

El primero trae el SDK y la integración con el arranque genérico de .NET; el segundo, el exporter OTLP; los tres últimos, instrumentación automática.

Y la configuración completa de las tres señales:

```csharp
using OpenTelemetry.Logs; using OpenTelemetry.Metrics;
using OpenTelemetry.Resources; using OpenTelemetry.Trace;

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(
            serviceName: "pedidos",
            serviceVersion: "1.4.0",
            serviceInstanceId: Environment.MachineName))
    .WithTracing(tracing => tracing
        .AddSource("MiTienda.Pedidos")          // tu ActivitySource, por nombre
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddMeter("MiTienda.Pedidos")           // tu Meter, por nombre
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddOtlpExporter())
    .WithLogging(logging => logging
        .AddOtlpExporter());
```

Con eso, el servicio ya manda trazas, métricas y logs por OTLP al endpoint configurado. Nótese que **no se ha escrito la URL en ningún sitio**: el exporter OTLP lee variables de entorno estándar, y eso es lo que permite desplegar el mismo binario en local y en producción.

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
```

`WithLogging()` está disponible desde la versión 1.9 del SDK. En versiones anteriores la parte de logs se configura aparte, sobre el pipeline de [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md):

```csharp
builder.Logging.AddOpenTelemetry(options => options.AddOtlpExporter());
```

## `service.name`: sin esto, todo se mezcla

`ConfigureResource` parece cosmético y no lo es. El **Resource** son los atributos que describen *quién* emite, y se adjuntan a cada span, cada métrica y cada log que sale del proceso. Sin él, el SDK inventa un valor por defecto:

```
❌ service.name = "unknown_service:MiTienda.Pedidos.Api"

✅ service.name  = "pagos"        service.instance.id = "pagos-7d4f9b"
   service.version = "1.4.0"      deployment.environment.name = "production"
```

Con cuatro servicios emitiendo a la vez y sin ese campo, el backend recibe cuatro montones de datos que no se pueden separar: no puedes filtrar "errores de `pagos`" ni comparar la latencia de `inventario` antes y después de un despliegue, porque no hay nada por lo que agrupar. `service.version` responde a "¿empezó con el despliegue de esta mañana?" y `service.instance.id`, a "¿falla en todas las instancias o solo en una?".

Todo esto se puede fijar también por entorno, sin tocar código:

```bash
OTEL_SERVICE_NAME=pagos
OTEL_RESOURCE_ATTRIBUTES=service.version=1.4.0,deployment.environment.name=production
```

## Lo que te dan gratis las instrumentaciones automáticas

Antes de escribir instrumentación propia conviene ver qué aparece solo, porque suele ser más de lo que uno espera:

| Paquete | Qué emite |
|---|---|
| `Instrumentation.AspNetCore` | Un span por petición entrante, con ruta, método y código de estado; el histograma `http.server.request.duration` |
| `Instrumentation.Http` | Un span por llamada saliente de `HttpClient` **y propaga la cabecera `traceparent`** |
| `Instrumentation.EntityFrameworkCore` | Un span por consulta a base de datos, con el texto del comando |
| `Instrumentation.Runtime` | Métricas de GC, memoria, hilos y excepciones del proceso |

La segunda fila es la más valiosa y la que menos se aprecia: es lo que hace que la traza **no se corte** cuando `pedidos` llama a `pagos`. Sin ella cada servicio generaría trazas sueltas sin relación. Los detalles de esa propagación están en [Tracing Distribuido](Tracing-Distribuido.md).

Con solo esos cuatro paquetes activados, confirmar el pedido **#4711** ya produce una traza como esta:

```
Trace 4bf92f35 — POST /api/pedidos/{id}/confirmar        2 340 ms
├─ POST http://pagos/api/cobros                          2 100 ms   ← el cuello de botella
│  └─ (span del servicio pagos)                          2 080 ms
├─ POST http://inventario/api/reservas                     110 ms
└─ SELECT MiTienda.Pedidos                                  40 ms
```

Nada de eso ha costado una línea de código de negocio.

## Instrumentación manual: `ActivitySource` y `Meter`

Lo automático cubre las fronteras técnicas (HTTP, base de datos). Lo que no cubre —y suele ser lo que interesa— son los pasos de negocio: *"validar stock"*, *"aplicar descuentos"*, *"llamar a la pasarela"*.

En .NET se instrumenta con los tipos del runtime. Un `ActivitySource` para trazas:

```csharp
using System.Diagnostics;

private static readonly ActivitySource Source = new("MiTienda.Pedidos");

public async Task ConfirmarAsync(Pedido pedido)
{
    using var activity = Source.StartActivity("pedidos.Confirmar");
    activity?.SetTag("pedido.id", pedido.Id);            // 4711
    activity?.SetTag("pedido.metodo_pago", "tarjeta");

    try { await CobrarAsync(pedido); }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.AddException(ex);
        throw;
    }
}
```

`Activity` es el equivalente en .NET a un *span* de OTel; el `using` lo cierra y calcula la duración. El `?` no sobra: si nadie está escuchando ese `ActivitySource`, `StartActivity` devuelve `null` y el coste de la instrumentación es prácticamente cero.

Y un `Meter` para métricas, con el mismo patrón:

```csharp
private static readonly Meter Meter = new("MiTienda.Pedidos", "1.0");
private static readonly Counter<long> PedidosConfirmados =
    Meter.CreateCounter<long>("pedidos.confirmados");

PedidosConfirmados.Add(1, new KeyValuePair<string, object?>("metodo_pago", "tarjeta"));
```

**El punto que conecta esto con OTel es el nombre.** El SDK no recoge todo lo que se emite en el proceso: recoge solo las fuentes que le has pedido explícitamente, por nombre exacto.

```csharp
❌ .AddSource("MiTienda.Pedidos.Api")   // el ActivitySource se llama "MiTienda.Pedidos"
✅ .AddSource("MiTienda.Pedidos")
```

Con el nombre mal escrito no hay error, no hay aviso: simplemente tus spans no aparecen mientras los automáticos sí, que es lo que hace tan desconcertante este fallo. Un truco que ahorra disgustos es declarar el nombre en una constante y usarla en los dos sitios. Para el detalle de qué instrumento elegir y cómo diseñar los atributos, [Métricas](Metricas.md).

## Exporters: a dónde salen los datos

Un *exporter* decide el destino, igual que un *sink* de [Serilog](Serilog.md). Se pueden encadenar varios.

**OTLP** es el que se usa en producción. Es el protocolo nativo de OpenTelemetry y hoy lo entienden casi todos los backends, así que evita necesitar un exporter distinto por herramienta:

```csharp
.AddOtlpExporter(options =>
{
    options.Endpoint = new Uri("http://otel-collector:4317");
    options.Protocol = OtlpExportProtocol.Grpc;
})
```

**Consola** (`.AddConsoleExporter()`), para depurar mientras montas todo esto. Es lo primero que hay que probar cuando "no llega nada": si en consola aparece y en el backend no, el problema es de red o de endpoint, no de instrumentación.

```
Activity.TraceId:     4bf92f3577b34da6a3ce929d0e0e4736
Activity.DisplayName: pedidos.Confirmar
Activity.Duration:    00:00:02.3401120
Activity.Tags:        pedido.id: 4711 / pedido.metodo_pago: tarjeta
Resource:             service.name: pedidos
```

**Prometheus**, solo para métricas y con un modelo distinto: en lugar de que la aplicación envíe, expone un endpoint `/metrics` que Prometheus consulta cada pocos segundos.

```csharp
.WithMetrics(metrics => metrics.AddPrometheusExporter());
app.MapPrometheusScrapingEndpoint();
```

Ese endpoint no debe quedar expuesto a internet; detrás de un [reverse proxy](../despliegue-en-vps/Reverse-Proxy-con-nginx-proxy.md) se limita a la red interna.

## El Collector: qué problema resuelve

El **Collector** es un proceso independiente —un contenedor— que recibe telemetría de varios servicios, la procesa y la reenvía a uno o varios destinos.

```
[pedidos] [pagos] [inventario] [emails] ──►  [ Collector ] ──┬──►  Jaeger      (trazas)
                                                             └──►  Prometheus  (métricas)
```

Resuelve tres cosas concretas: cambiar de backend sin redesplegar ni un servicio, aplicar reglas comunes en un solo sitio (filtrar rutas de *health check*, borrar datos personales, añadir atributos del entorno) y que las aplicaciones no dependan de que el backend esté disponible.

**Y cuándo no lo necesitas: al empezar.** Exportar directamente desde la aplicación al backend funciona perfectamente con uno o dos servicios, y es una pieza menos que desplegar, vigilar y arreglar de madrugada. Añádelo cuando aparezca alguno de los tres problemas de arriba, no antes.

Una configuración mínima real, `otel-collector-config.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }

processors:
  batch: { timeout: 5s }
  memory_limiter: { check_interval: 1s, limit_mib: 400 }

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls: { insecure: true }
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

La estructura se repite siempre: **receivers** (por dónde entra), **processors** (qué se le hace en medio) y **exporters** (por dónde sale). Y la clave está en `service.pipelines`: definir un componente arriba no lo activa; solo cuenta si aparece en una pipeline. Es el fallo número uno al configurar un Collector por primera vez —un exporter declarado y nunca referenciado, y nadie recibe nada.

`memory_limiter` no es opcional en producción: sin él, un pico de telemetría puede hacer que el Collector consuma toda la memoria de la máquina y arrastre a los servicios que vigila.

## Convenciones semánticas

OTel publica un catálogo de nombres estándar para atributos y métricas. No es burocracia: es lo que hace que un panel funcione sin retocarlo cuando cambias de backend, y que las herramientas sepan interpretar lo que envías.

```csharp
❌ activity?.SetTag("metodo", "POST");
   activity?.SetTag("statusCode", 500);

✅ activity?.SetTag("http.request.method", "POST");
   activity?.SetTag("http.response.status_code", 500);
```

Con los nombres estándar, Jaeger colorea los spans con error, Grafana rellena sus paneles de HTTP sin configuración y el mismo dashboard sirve para los cuatro servicios. Con nombres inventados, todo eso hay que construirlo a mano.

La regla práctica: **si el dato ya tiene nombre en el catálogo, úsalo**; para lo tuyo, inventa un prefijo propio y sé consistente (`pedido.id`, `pedido.metodo_pago`, nunca `orderId` en un sitio y `order_id` en otro).

## Logs: cómo convive con Serilog

Es la señal que más dudas genera, porque parece que OTel y Serilog compiten. No compiten. En .NET todo el logging pasa por `ILogger`, y OTel se engancha como un *proveedor* más de [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md), igual que la consola o que Serilog. Hay dos montajes razonables:

| Montaje | Cómo | Cuándo |
|---|---|---|
| Solo OTel | `WithLogging().AddOtlpExporter()` | Proyecto nuevo, todo (logs, métricas, trazas) al mismo backend por OTLP |
| Serilog + sink OTLP | `Serilog.Sinks.OpenTelemetry` | Ya usas Serilog y quieres conservar sus *enrichers*, `LogContext` y sus otros destinos |

El segundo montaje, con el paquete `Serilog.Sinks.OpenTelemetry`:

```csharp
builder.Services.AddSerilog((services, config) => config
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.OpenTelemetry(options =>
    {
        options.Endpoint = "http://otel-collector:4318/v1/logs";
        options.ResourceAttributes = new Dictionary<string, object>
            { ["service.name"] = "pedidos" };
    }));
```

Lo que hay que evitar es tener las dos rutas activas a la vez mandando al mismo sitio: acabas con cada log duplicado en el backend. Elige una.

En ambos montajes el `traceId` viaja solo. Como los logs se emiten dentro de la `Activity` en curso, el proveedor de OTel adjunta `trace_id` y `span_id` a cada evento sin que haya que escribirlos. Eso es exactamente lo que permite saltar de un log de error a la traza completa que lo rodea, que es el objetivo de todo esto ([Logging Estructurado](Logging-Estructurado.md)).

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| Aparecen los spans automáticos pero no los míos | Falta `AddSource("...")`, o el nombre no coincide **exactamente** con el del `ActivitySource` |
| Aparecen las métricas automáticas pero no las mías | Lo mismo con `AddMeter("...")` |
| Todo llega como `unknown_service:...` | Falta `ConfigureResource(...AddService(...))` o `OTEL_SERVICE_NAME` |
| No llega absolutamente nada al backend | Puerto/protocolo cruzados: `:4317` es gRPC, `:4318` es HTTP |
| Con `:4318` da 404 | En HTTP el endpoint lleva ruta: `/v1/traces`, `/v1/metrics`, `/v1/logs` |
| El Collector arranca bien pero no reenvía | El exporter está declarado pero no incluido en `service.pipelines` |
| Desde un contenedor no conecta a `localhost:4317` | `localhost` es el propio contenedor; usa el nombre del servicio de la red Docker |
| La traza se corta al saltar a otro servicio | Falta `AddHttpClientInstrumentation()`, o el salto es por cola y no propaga `traceparent` |
| Faltan los últimos datos al parar el proceso | El exportador por lotes no llegó a vaciar; hace falta apagado ordenado |
| Cada log aparece dos veces | Serilog con sink OTLP **y** `WithLogging()` activos a la vez |

Cuando no sabes en qué punto se pierde, el orden de diagnóstico es siempre el mismo: primero `AddConsoleExporter()` para confirmar que la telemetría se genera, y solo después mirar red, endpoint y Collector.

## Buenas prácticas avanzadas

- **Empieza sin Collector y añádelo cuando duela algo concreto.** Exportar directo de la aplicación al backend es válido en producción y una pieza menos que mantener. El Collector se justifica cuando necesitas cambiar de destino sin redesplegar, aplicar reglas comunes a varios servicios o desacoplar tus apps de la disponibilidad del backend — no por completitud arquitectónica.
- **Configura endpoints y `Resource` por variables de entorno, nunca en código.** Las `OTEL_*` son estándar y están soportadas por todos los SDK, así que el mismo binario sirve para local, staging y producción. Un endpoint escrito a mano en `Program.cs` obliga a recompilar para cambiar de backend, que es justo lo que OTel venía a evitar.
- **Trata los nombres de spans, métricas y atributos como un contrato público.** Renombrarlos rompe en silencio paneles y alertas, y nadie se entera hasta que hace falta esa alerta. Usa las convenciones semánticas donde existan y, si tienes que cambiar un nombre propio, emite ambos durante una temporada.
- **Vigila la cardinalidad también en los spans.** El nombre de un span debe ser la plantilla (`GET /api/pedidos/{id}`), nunca la URL con el identificador dentro; con la URL real, el backend ve un millón de operaciones distintas y la agrupación por operación deja de funcionar. El identificador va en un atributo (`pedido.id`), que para eso está.
- **Comprueba que un backend caído no tumba la aplicación.** El exportador por lotes con timeout acotado es el comportamiento correcto: si el Collector no responde, se descarta telemetría y las peticiones siguen atendiéndose. Un exportador síncrono y sin timeout convierte una caída de la herramienta de observabilidad en una caída del servicio, que es el peor desenlace posible.

## Recursos didácticos

- [OpenTelemetry Demo](https://opentelemetry.io/docs/demo/) — una tienda online completa con microservicios en varios lenguajes, instrumentada de verdad y con Jaeger, Prometheus y Grafana incluidos. Un `docker compose up` y ya puedes navegar trazas reales que cruzan servicios; es lo más parecido a ver el resultado final antes de instrumentar lo tuyo.
- [Panel de .NET Aspire en modo independiente](https://learn.microsoft.com/dotnet/aspire/fundamentals/dashboard/standalone) — un único contenedor que actúa como visor OTLP local: apuntas tu aplicación a su puerto y ves logs, métricas y trazas correlacionados sin montar ningún backend. Es la forma más rápida de comprobar que tu instrumentación funciona.
- [Convenciones semánticas de OpenTelemetry](https://opentelemetry.io/docs/specs/semconv/) — el catálogo de nombres estándar. Conviene consultarlo *antes* de inventar un atributo: casi siempre el dato que quieres registrar ya tiene nombre oficial, y usarlo es lo que hace que los paneles de terceros funcionen sin retoques.

---

*En resumen: OpenTelemetry te deja instrumentar una vez con la API del propio runtime y decidir el destino solo con configuración — configura el `Resource` desde el primer día, registra tus fuentes con `AddSource`/`AddMeter` con el nombre exacto, y deja el Collector para cuando de verdad haga falta.*
