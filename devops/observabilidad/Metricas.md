# Métricas

## ¿Qué es?

Una métrica es un número medido a lo largo del tiempo que resume el comportamiento de un sistema: cuántas peticiones ha recibido, cuánto tarda en responder, cuánta memoria consume. A diferencia de un log, que describe un evento puntual con detalle, una métrica es un dato **agregado** y baratísimo de conservar durante meses.

## ¿Por qué existe?

Imagina responder a "¿cuántas peticiones por segundo recibimos la semana pasada?" a partir de los logs. Habría que guardar una entrada por cada petición durante toda la semana —millones de registros, gigabytes— y después contarlas. Para obtener un número.

Las métricas invierten el planteamiento: en lugar de guardar cada evento y contar después, **se cuenta al vuelo y se guarda solo el recuento**. En lugar de un millón de registros, un número por intervalo. La diferencia de coste es de varios órdenes de magnitud, y es lo que permite conservar años de historia y comparar el rendimiento de hoy con el del mismo día del mes pasado.

> Piensa en la diferencia entre las cámaras de un peaje —registran cada coche con su matrícula, el equivalente a un log— y el contador de vehículos por hora del ayuntamiento —solo le importa el total, el equivalente a una métrica—. El segundo ocupa una millonésima parte y basta para casi todas las preguntas sobre tráfico.

## ¿Cuándo y para qué se usa?

Para dos cosas que los logs no pueden hacer bien:

- **Alertar.** "Avísame si la tasa de error supera el 5 %" necesita un número continuo, no una búsqueda de texto sobre millones de líneas.
- **Ver tendencias.** "¿La latencia ha empeorado desde el despliegue?" requiere comparar dos periodos, y eso con logs es carísimo.

Lo que **no** hacen es explicar un caso concreto. Una métrica dice que el 2 % de las peticiones falla; para saber cuál falló y por qué hacen falta [logs](Logging-Estructurado.md) y [trazas](Tracing-Distribuido.md).

---

## Los tipos de instrumento

Cuatro formas de medir, y elegir mal complica el análisis después.

### Contador: solo sube

Cuenta eventos acumulados desde que arrancó el proceso.

```csharp
private static readonly Meter Meter = new("MiTienda.Pedidos", "1.0");
private static readonly Counter<long> PedidosCreados =
    Meter.CreateCounter<long>("pedidos.creados", unit: "{pedido}",
        description: "Total de pedidos creados");

PedidosCreados.Add(1, new KeyValuePair<string, object?>("estado", "confirmado"));
```

El valor absoluto de un contador casi nunca interesa —"llevamos 4 371 892 pedidos desde el último reinicio" no dice nada—. Lo que interesa es **su tasa de cambio**, que calcula la herramienta de consulta:

```promql
rate(pedidos_creados_total[5m])
```

Tratar un contador como si fuera un valor instantáneo es el error más común al empezar. Y hay un motivo por el que existe `rate()` en lugar de restar valores: cuando el proceso se reinicia, el contador vuelve a cero, y `rate()` detecta ese salto y lo maneja. Una resta manual daría un número negativo enorme.

### Gauge: sube y baja

Un valor en un instante concreto: memoria usada ahora, conexiones activas, mensajes esperando en una cola.

```csharp
Meter.CreateObservableGauge("cola.pendientes",
    () => colaDePedidos.Count,
    unit: "{mensaje}",
    description: "Pedidos esperando procesamiento");
```

`ObservableGauge` recibe una función que el sistema **invoca cuando toca medir**, en lugar de que tú notifiques cada cambio. Es lo correcto para valores que se pueden consultar en cualquier momento; forzar un contador aquí no funcionaría, porque un gauge baja.

### Histograma: la distribución

En vez de un número, agrupa las observaciones en cubos para poder calcular percentiles.

```csharp
private static readonly Histogram<double> DuracionPedido =
    Meter.CreateHistogram<double>("pedido.duracion", unit: "s",
        description: "Tiempo de confirmación de un pedido");

var sw = Stopwatch.StartNew();
await ConfirmarPedidoAsync(pedido);
DuracionPedido.Record(sw.Elapsed.TotalSeconds,
    new KeyValuePair<string, object?>("metodo_pago", "tarjeta"));
```

Es el instrumento correcto para latencia, **siempre**, y el motivo merece su propia sección más abajo.

### UpDownCounter: sube y baja, pero lo notificas tú

A medio camino entre contador y gauge: acumula, pero admite valores negativos.

```csharp
private static readonly UpDownCounter<long> PedidosEnProceso =
    Meter.CreateUpDownCounter<long>("pedidos.en_proceso");

PedidosEnProceso.Add(1);    // al empezar
// ...
PedidosEnProceso.Add(-1);   // al terminar
```

Se usa cuando el valor no se puede consultar de golpe pero sí sabes cuándo cambia.

| Instrumento | Cuándo | Ejemplo |
|---|---|---|
| `Counter` | Solo crece, cuentas eventos | Peticiones totales, errores totales |
| `Histogram` | Distribución de una medida | Latencia, tamaño de respuesta |
| `ObservableGauge` | Valor consultable en cualquier momento | Memoria, tamaño de cola |
| `UpDownCounter` | Sube y baja, lo notificas al ocurrir | Trabajos en curso |

## Por qué la media miente

Esta es la razón de que la latencia se mida con histogramas y no con un promedio, y merece verse con números.

Supón mil peticiones: 950 tardan 100 ms y 50 tardan 5 segundos.

```
Media = (950 × 0,1 + 50 × 5) / 1000 = 0,34 s
```

340 milisegundos de media parece perfectamente razonable. Y sin embargo **una de cada veinte personas está esperando cinco segundos**. La media ha diluido el problema hasta hacerlo invisible.

Los percentiles sí lo reflejan:

```
p50  (mediana)  = 0,10 s     la mitad va bien
p95             = 0,10 s     el 95 % también
p99             = 5,00 s     el 1 % peor sufre 5 segundos
```

Por eso los objetivos de servicio se escriben siempre sobre percentiles: *"el 99 % de las peticiones responde en menos de 500 ms"*. Y por eso conviene mirar p99 y no solo p95: con mucho tráfico, ese 1 % son miles de personas al día.

Un matiz que se descubre tarde: **los percentiles no se pueden promediar**. La media de los p95 de tres instancias no es el p95 del conjunto. Para agregar correctamente hay que sumar los cubos de los histogramas y calcular el percentil sobre el total, que es justo lo que hace `histogram_quantile()` en Prometheus. Si tu panel promedia percentiles entre instancias, el número que muestra no significa nada.

## Etiquetas y el peligro de la cardinalidad

Las **etiquetas** trocean una métrica por dimensiones:

```csharp
PedidosCreados.Add(1,
    new KeyValuePair<string, object?>("estado", "confirmado"),
    new KeyValuePair<string, object?>("metodo_pago", "tarjeta"));
```

Ahora se puede preguntar "pedidos confirmados pagados con tarjeta" sin definir una métrica nueva.

Y aquí está el error que tumba sistemas de métricas y dispara facturas. **Cada combinación distinta de valores de etiqueta crea una serie temporal independiente.** Con 4 estados y 3 métodos de pago:

```
4 × 3 = 12 series. Perfectamente asumible.
```

Ahora alguien añade el identificador del pedido, con la mejor intención:

```csharp
// ☠️ NUNCA
PedidosCreados.Add(1, new KeyValuePair<string, object?>("pedido_id", pedido.Id));
```

```
4 × 3 × (un millón de pedidos) = 12 000 000 de series
```

Doce millones de series temporales, cada una con su historial. Esto no degrada el sistema de métricas: **lo tumba**, o genera una factura de varios miles de euros. Y tiene nombre propio, *cardinality explosion*, porque le ha pasado a todo el mundo al menos una vez.

La regla es simple y no admite excepciones: **una etiqueta solo puede tomar un conjunto pequeño y acotado de valores conocidos de antemano.**

| ✅ Válido como etiqueta | ☠️ Nunca como etiqueta |
|---|---|
| `estado` (4 valores) | `pedido_id` |
| `metodo_pago` (3) | `usuario_id` |
| `endpoint` (~50, con plantilla) | `email`, `ip` |
| `codigo_estado` (~10) | `url` completa con parámetros |

El `endpoint` tiene truco: hay que usar la **plantilla de ruta** (`/api/pedidos/{id}`), no la URL real (`/api/pedidos/4711`). Con la URL real vuelves a tener una serie por pedido. La instrumentación automática de ASP.NET Core ya lo hace bien; el peligro está en la manual.

Y la contrapartida: esos identificadores **sí** deben estar en los logs y las trazas, que están diseñados para el detalle. Cada señal para lo suyo.

## Exponer las métricas

Definidas las métricas, alguien tiene que recogerlas. El modelo más extendido es el de Prometheus: la aplicación expone un endpoint y el sistema de métricas lo consulta cada pocos segundos.

Con [OpenTelemetry](OpenTelemetry.md) en .NET:

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore
```

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddMeter("MiTienda.Pedidos")              // tu Meter, por nombre
        .AddAspNetCoreInstrumentation()            // peticiones HTTP entrantes
        .AddHttpClientInstrumentation()            // llamadas salientes
        .AddRuntimeInstrumentation()               // GC, hilos, memoria
        .AddPrometheusExporter());

var app = builder.Build();
app.MapPrometheusScrapingEndpoint();               // expone /metrics
```

`AddMeter("MiTienda.Pedidos")` es imprescindible y se olvida a menudo: si el nombre no coincide **exactamente** con el del `Meter` que creaste, tus métricas propias no se exportan y las automáticas sí, lo que despista bastante.

Lo que sirve `/metrics` es texto plano:

```
# HELP pedidos_creados_total Total de pedidos creados
# TYPE pedidos_creados_total counter
pedidos_creados_total{estado="confirmado",metodo_pago="tarjeta"} 1847
pedidos_creados_total{estado="cancelado",metodo_pago="tarjeta"} 23

# TYPE pedido_duracion_seconds histogram
pedido_duracion_seconds_bucket{le="0.1"} 4210
pedido_duracion_seconds_bucket{le="0.5"} 4890
pedido_duracion_seconds_bucket{le="+Inf"} 4912
pedido_duracion_seconds_sum 743.2
pedido_duracion_seconds_count 4912
```

Los `_bucket` son los cubos del histograma: "4 210 observaciones tardaron 0,1 s o menos". De ahí salen los percentiles.

**Ese endpoint no debe ser público.** Revela la estructura interna del sistema y su volumen de negocio. En un despliegue con [reverse proxy](../despliegue-en-vps/Reverse-Proxy-con-nginx-proxy.md), lo correcto es dejarlo accesible solo desde la red interna.

## Qué medir: RED y USE

Para no caer en el panel de cien gráficas que nadie mira, hay dos catálogos que funcionan.

**RED**, para cada servicio que atiende peticiones:

```promql
# Rate — peticiones por segundo
rate(http_server_request_duration_seconds_count[5m])

# Errors — proporción de fallos
rate(http_server_request_duration_seconds_count{http_response_status_code=~"5.."}[5m])
  / rate(http_server_request_duration_seconds_count[5m])

# Duration — el percentil 99
histogram_quantile(0.99, rate(http_server_request_duration_seconds_bucket[5m]))
```

**USE**, para cada recurso compartido (CPU, disco, pool de conexiones, cola): utilización, saturación y errores.

Con esas dos plantillas aplicadas a cada servicio y cada recurso cubres la mayoría de los incidentes reales, sin inventar nada.

## Alertar sobre métricas

Una métrica sin alerta es una gráfica que alguien mirará cuando ya sea tarde. Pero alertar mal es peor que no alertar, porque entrena al equipo a ignorar los avisos.

Dos reglas que evitan casi todo el ruido:

**Alerta sobre síntomas, no sobre causas.** "La tasa de error supera el 5 %" afecta a usuarios reales. "La CPU está al 90 %" puede ser perfectamente normal en un proceso por lotes. La primera merece despertar a alguien; la segunda, como mucho, un panel.

**Exige que el problema persista.** Un pico de dos segundos se resuelve solo; cinco minutos de degradación sostenida no.

```yaml
- alert: TasaDeErrorAlta
  expr: |
    rate(http_server_request_duration_seconds_count{http_response_status_code=~"5.."}[5m])
      / rate(http_server_request_duration_seconds_count[5m]) > 0.05
  for: 5m                    # ← lo que filtra los picos transitorios
  annotations:
    summary: "Más del 5% de errores en {{ $labels.service }} desde hace 5 minutos"
```

## Buenas prácticas avanzadas

- **Trata el nombre de cada métrica y etiqueta como un contrato público.** Renombrar una métrica rompe en silencio todos los paneles y alertas que dependían de ella, y nadie se entera hasta que hace falta esa alerta. Sigue las convenciones semánticas de OpenTelemetry (`http.server.request.duration`) en lugar de inventar nombres, y si tienes que cambiar uno, publica ambos durante una temporada.
- **Mide el negocio, no solo la infraestructura.** CPU y memoria hacen falta, pero "pedidos perdidos por fallo de pago" avisa de un problema real mucho antes que cualquier alerta de recursos — y hay incidentes en los que toda la infraestructura está en verde mientras el negocio se cae. Un contador por cada resultado que le importe a alguien de negocio suele ser la métrica más valiosa del sistema.
- **Ajusta los cubos del histograma a tu latencia real.** Los cubos por defecto están pensados para un caso genérico; si tus peticiones tardan entre 5 y 50 ms y el primer cubo es 100 ms, **todas** caen en el mismo y el p99 no distingue nada. Definir límites acordes al rango que mides es lo que convierte el histograma en información en lugar de en una línea plana.
- **Vigila la cardinalidad como vigilas el disco.** El número de series activas es una métrica más y merece su propia alerta: crece de golpe cuando alguien añade una etiqueta sin pensar, y para cuando llega la factura llevas semanas pagándola. Un aviso al superar un umbral de series detecta el problema el mismo día del despliegue que lo introdujo.
- **Recuerda que un contador que deja de incrementarse no emite nada.** Si alertas sobre "cero pedidos en la última hora", la ausencia total de datos —porque el servicio está caído y no expone nada— puede no disparar la alerta según cómo esté escrita la consulta. Combinar la alerta de negocio con una de *"el servicio no responde al scrape"* cubre el caso en que el fallo es tan grave que ni siquiera puede informar de él.

## Recursos didácticos

- [Prometheus + Grafana con Docker](https://prometheus.io/docs/prometheus/latest/getting_started/) — levantar los dos en contenedores, apuntarlos a una aplicación .NET propia y construir el primer panel es la forma más rápida de que contadores, histogramas y `rate()` dejen de ser abstractos.
- [PromLabs PromQL tutorial](https://demo.promlabs.com/) — un entorno interactivo con datos reales donde escribir consultas PromQL y ver el resultado al instante. Es donde se entiende de verdad por qué `rate()` es necesario y cómo funciona `histogram_quantile()`.
- [Google SRE Book, *Monitoring Distributed Systems*](https://sre.google/sre-book/monitoring-distributed-systems/) — las cuatro señales doradas y, sobre todo, el capítulo sobre por qué la mayoría de las alertas sobran y cómo decidir cuáles merecen despertar a alguien.

---

*En resumen: las métricas son la forma barata de vigilar un sistema en el tiempo — usa histogramas para la latencia porque la media miente, y no pongas jamás un identificador como etiqueta.*
