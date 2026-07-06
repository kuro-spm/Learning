# Métricas

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Una métrica es un número medido a lo largo del tiempo que resume el comportamiento de un sistema: cuántas peticiones ha recibido, cuánta memoria usa, cuánto tarda en responder. A diferencia de un log, que describe un evento puntual con detalle, una métrica es un dato agregado y barato de almacenar durante mucho tiempo.

## ¿Por qué existe?

Guardar un log por cada petición HTTP durante meses para saber "cuántas peticiones recibimos por segundo la semana pasada" es carísimo e innecesario: no hace falta el detalle de cada petición individual, solo el número agregado. Las métricas existen para responder ese tipo de preguntas de forma barata: un contador que sube, un número que sube y baja, sin guardar el rastro completo de cada evento que lo generó.

> Piensa en la diferencia entre las cámaras de un peaje (registran cada coche, matrícula incluida — el equivalente a un log) y el contador de vehículos por hora del ayuntamiento (solo le importa el número total — el equivalente a una métrica). El segundo pesa mucho menos y basta para casi todas las preguntas sobre tráfico.

## ¿Cuándo y para qué se usa?

Para vigilar la salud y el rendimiento de un sistema en producción de forma continua: cuántas peticiones por segundo recibe la API de una tienda online, cuántos pedidos fallan por minuto, cuánto tarda en responder el percentil 95 de las peticiones, cuánta memoria consume un servicio a lo largo del día. Son la base de los dashboards y de las alertas automáticas ("avisa si el error rate supera el 5%").

## Lo mínimo que necesitas saber

**1. Contador (*counter*): un número que solo sube**

Cuenta eventos acumulados desde que arrancó el proceso: total de peticiones, total de pedidos creados, total de errores.

```csharp
var pedidosCreados = meter.CreateCounter<long>("pedidos_creados_total");
pedidosCreados.Add(1, new KeyValuePair<string, object>("estado", "confirmado"));
```

**2. Gauge: un número que sube y baja**

Representa un valor en un instante concreto: memoria usada ahora mismo, número de conexiones activas, elementos en una cola.

```csharp
var conexionesActivas = meter.CreateObservableGauge("conexiones_activas",
    () => connectionPool.ActiveConnections);
```

**3. Histograma: la distribución de un valor**

En vez de un solo número, agrupa las observaciones en "cubos" para poder calcular percentiles: no solo la media de la duración de las peticiones, sino cuánto tarda el 95% de ellas.

```csharp
var duracionPeticion = meter.CreateHistogram<double>("http_request_duration_seconds");
duracionPeticion.Record(0.245); // esta petición ha tardado 245 ms
```

**4. Las métricas son series temporales**

Cada valor se guarda junto a la marca de tiempo en la que se midió, lo que permite dibujar su evolución y compararla con la de otros momentos (ahora frente a hace una semana).

```
http_requests_total{status="500"}
10:00 → 3
10:05 → 3
10:10 → 47   <- algo se ha roto aquí
```

**5. Etiquetas (*labels*/*tags*): trocear la métrica**

Permiten desglosar el mismo número por dimensiones: peticiones por endpoint, por código de estado, por servicio. Cada combinación de valores de etiquetas genera una serie temporal distinta.

```csharp
pedidosCreados.Add(1, new("estado", "confirmado"), new("metodoPago", "tarjeta"));
```

## Lo que NO hace

- **No cuenta la historia completa de un evento concreto** — para saber "por qué" falló un pedido en particular hace falta un log o una traza, no una métrica.
- **No sirve para depurar un caso individual** — una métrica te dice que el 2% de las peticiones fallan, no cuál falló ni por qué.
- **No es gratis multiplicar etiquetas** — cada combinación distinta de valores de etiquetas crea una serie temporal nueva que hay que almacenar.
- **No sustituye a las alertas por sí sola** — la métrica es el dato; alguien tiene que definir sobre qué umbral se dispara una alerta.

## Buenas prácticas avanzadas

- **Nunca uses un valor de alta cardinalidad como etiqueta** — `orderId`, `userId` o un UUID como valor de una etiqueta multiplican las series temporales por cada pedido o usuario que exista, lo que puede tumbar el sistema de métricas o disparar su coste. Esos identificadores van en logs o trazas, no en etiquetas de métricas.
- **Usa histogramas para latencia, nunca solo la media** — la media esconde a los usuarios que peor lo pasan: si el 95% de las peticiones tarda 100 ms y el 5% tarda 5 segundos, la media parece razonable pero una de cada veinte personas sufre una espera enorme. Los percentiles (p95, p99) sacados de un histograma sí lo reflejan.
- **Los contadores nunca bajan, ni siquiera al reiniciar el proceso lógicamente** — si necesitas saber "cuántos ocurrieron en los últimos 5 minutos", eso se calcula con la tasa de cambio del contador (`rate()` en Prometheus), no leyendo el contador directamente; tratar un contador como si fuera un gauge es un error común al empezar.
- **Diseña las métricas alrededor de preguntas del negocio, no solo de infraestructura** — CPU y memoria son necesarias, pero "cuántos pedidos se están perdiendo por fallos de pago" avisa de un problema real mucho antes que una alerta de CPU. Las métricas más valiosas suelen mapear directamente a algo que le importa al negocio.

---

*En resumen: las métricas son la forma barata de vigilar un sistema en el tiempo — no cuentan la historia completa de cada evento, pero dicen cuánto y con qué frecuencia pasa algo, y son la base de dashboards y alertas.*
