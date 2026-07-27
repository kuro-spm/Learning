# Observabilidad

## ¿Qué es?

La observabilidad es la capacidad de entender qué está pasando dentro de un sistema en producción a partir de lo que ese sistema expone hacia fuera —logs, métricas y trazas—, sin modificarlo ni conectarle un depurador.

## ¿Por qué existe?

El monitoring clásico vigila señales que ya sabes que importan: ¿está el servidor encendido?, ¿responde el *health check*?, ¿ha saltado la alerta de CPU al 90 %? Funciona bien mientras los fallos se parezcan a los que ya has visto.

El problema aparece con sistemas distribuidos. Una tienda online con servicios de pedidos, pagos, inventario y emails, cada uno desplegado por separado. Cuando un pedido tarda 12 segundos en confirmarse, ningún panel predefinido tiene la respuesta, porque **nadie anticipó esa pregunta**. Y no puedes añadir un `Console.WriteLine` y reproducirlo: está pasando ahora, en producción, con datos reales.

La distinción que lo resume: el monitoring responde a **incógnitas conocidas** (sé qué puede fallar y lo vigilo); la observabilidad responde a **incógnitas desconocidas** (algo va mal de una forma que nadie previó y necesito investigarlo con lo que ya está registrado).

> Piensa en el salpicadero de un coche frente al ordenador de diagnóstico del taller. El salpicadero enseña unas pocas señales fijas: velocidad, gasolina, revoluciones. El ordenador del taller puede responder preguntas nuevas sobre el motor, porque accede a datos en crudo que nadie decidió de antemano qué forma tendrían.

## ¿Cuándo y para qué se usa?

Se vuelve imprescindible en cuanto el sistema deja de ser "una aplicación en un servidor": varios servicios, colas de mensajes, varias instancias detrás de un balanceador. Las preguntas que aparecen entonces son de este tipo:

- ¿Por qué este pedido concreto se quedó a medias entre pagos e inventario?
- ¿Por qué un endpoint se ha vuelto lento **solo para algunos** usuarios?
- ¿Qué ha cambiado desde el despliegue de esta mañana?
- ¿A qué servicio está llamando esto que nadie recuerda haber añadido?

Ninguna se responde con una alerta de CPU.

---

## Los tres pilares

Se llaman pilares porque cada uno responde a una pregunta distinta sobre el mismo sistema. No compiten: se complementan, y usar el equivocado hace el trabajo mucho más difícil de lo necesario.

### Logs: qué pasó

Un registro de un evento puntual, con todo el detalle de su contexto.

```json
{
  "timestamp": "2026-07-27T10:15:32.418Z",
  "level": "Error",
  "service": "pagos",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "pedidoId": 4711,
  "motivo": "fondos_insuficientes",
  "message": "Pago rechazado por el banco"
}
```

Responden preguntas del tipo *"¿qué le pasó exactamente al pedido 4711?"*. Son la señal más rica en detalle y la más cara de almacenar: cada evento ocupa espacio y en un sistema con tráfico son millones al día.

### Métricas: cuánto y con qué frecuencia

Números agregados a lo largo del tiempo. No guardan el detalle de cada evento, solo el recuento.

```
http_requests_total{service="pagos", status="500"}   42
http_request_duration_seconds{service="pagos", quantile="0.95"}   1.8
```

Responden *"¿cuántos pagos están fallando por minuto?"* o *"¿ha empeorado la latencia desde ayer?"*. Son baratísimas de guardar durante meses —un número por intervalo, no un registro por evento—, y por eso son la base de los paneles y las alertas.

### Trazas: por dónde pasó

El recorrido completo de una petición concreta a través de todos los servicios que la atendieron.

```
Trace 4bf92f35 — "POST /api/pedidos/4711/confirmar"        (2 340 ms)
├─ pedidos.CrearPedido                                       120 ms
│  └─ pedidos.GuardarEnBD                                     40 ms
├─ pagos.Cobrar                                             2 100 ms   ← aquí está el problema
│  └─ HTTP POST api.pasarela-pago.com                       2 080 ms
└─ inventario.ReservarStock                                  110 ms
```

Responden *"¿en qué tramo se fue el tiempo?"*. Esa tabla de arriba localiza el cuello de botella en dos segundos de lectura; sin trazas, la misma conclusión sale de cruzar los logs de tres servicios a mano.

## Qué señal usar para cada pregunta

Esta es la tabla que conviene tener clara antes de instrumentar nada, porque el error caro es usar la señal equivocada:

| La pregunta | La señal | Por qué |
|---|---|---|
| ¿Está fallando algo ahora mismo? | Métricas | Baratas, continuas, sirven para alertar |
| ¿Cuántos y con qué frecuencia? | Métricas | Están agregadas de serie |
| ¿En qué tramo se va el tiempo? | Trazas | Es lo único que ve el recorrido completo |
| ¿Qué le pasó a este caso concreto? | Logs | Es lo único con el detalle del evento |
| ¿Por qué falló? | Logs (llegando desde la traza) | La traza localiza, el log explica |

El recorrido típico de una investigación real usa las tres en ese orden: **una alerta de métrica** avisa de que la tasa de error subió, **una traza** enseña qué servicio y qué tramo, y **el log** de ese tramo cuenta qué excepción saltó.

Y el error más común y más caro: usar logs para contar. Escribir un `Information` por cada petición para después contar líneas y saber cuántas hubo funciona hasta que el volumen se multiplica; entonces cuesta una fortuna en almacenamiento y la consulta tarda minutos. Eso es una métrica.

## El hilo que une los tres

Tres señales sueltas valen mucho menos que tres señales correlacionadas. Lo que las cose es un identificador que viaja con la petición: el **`traceId`**.

Se genera cuando la petición entra al sistema, viaja de servicio en servicio en una cabecera estándar y se incluye en cada log y cada span. Eso permite el salto que hace útil todo lo demás: ver un error en un log y abrir con un clic la traza completa que lo rodea, o al revés.

En .NET, el `traceId` actual está disponible sin configurar nada:

```csharp
using System.Diagnostics;

logger.LogError("Pago rechazado para el pedido {PedidoId} (traza {TraceId})",
    pedido.Id, Activity.Current?.TraceId.ToString());
```

Aunque escribirlo a mano en cada log es innecesario: lo habitual es que el sistema de logging lo añada solo a todo lo que se registre dentro de la petición, con los *scopes* de [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md) o los *enrichers* de [Serilog](Serilog.md).

**Si solo vas a hacer una cosa de toda esta guía, que sea esta.** Un sistema con logs, métricas y trazas sin correlacionar obliga a cruzar marcas de tiempo a mano; con el `traceId` en todo, la investigación se vuelve navegación.

## Por dónde empezar

El orden importa, porque cada paso hace útil al siguiente y el esfuerzo es creciente:

1. **Logs estructurados con `traceId`.** Es lo más barato y lo que más rinde de inmediato. Sin esto, lo demás no se puede conectar.
2. **Las cuatro métricas básicas** de cada servicio: peticiones por segundo, tasa de error, latencia (p95, no la media) y saturación (CPU/memoria/cola). Con eso ya puedes alertar.
3. **Trazas con instrumentación automática.** En .NET, activar [OpenTelemetry](OpenTelemetry.md) da las trazas de HTTP y base de datos sin escribir código propio.
4. **Instrumentación manual** de los pasos que le importan al negocio.

Saltar al paso 4 antes del 1 es el camino habitual y el que produce sistemas con mucha telemetría y poca capacidad de investigar.

## Las señales que merece la pena vigilar

Hay dos catálogos clásicos que evitan la parálisis de "¿qué mido?".

**RED**, para servicios que atienden peticiones:

- **R**ate — peticiones por segundo
- **E**rrors — cuántas fallan
- **D**uration — cuánto tardan (distribución, no media)

**USE**, para recursos (CPU, disco, pool de conexiones):

- **U**tilization — qué porcentaje del tiempo está ocupado
- **S**aturation — cuánto trabajo espera en cola
- **E**rrors — errores del propio recurso

Con RED en cada servicio y USE en cada recurso compartido tienes cubierto el 80 % de los incidentes reales. Y sobre esas señales se definen los **SLI** (el indicador: "el 99 % de las peticiones responde en menos de 500 ms") y los **SLO** (el objetivo comprometido). Definir eso primero es lo que evita el panel de cien gráficas que nadie mira.

## Lo que cuesta

La observabilidad no es gratis y conviene saber por dónde se escapa el dinero antes de recibir la factura:

| Coste | De dónde sale | Cómo se controla |
|---|---|---|
| Almacenamiento de logs | Volumen × retención | Niveles adecuados, retención corta para `Debug` |
| Series de métricas | Cardinalidad de las etiquetas | No usar identificadores como etiqueta |
| Trazas | Volumen de peticiones | *Sampling* con criterio |
| Rendimiento | Instrumentación en rutas calientes | `IsEnabled`, `[LoggerMessage]`, sinks asíncronos |

De todos, el que más sorprende es la **cardinalidad**. Añadir una etiqueta `pedidoId` a una métrica no crea una serie: crea *una serie por cada pedido que exista*. Un millón de pedidos son un millón de series temporales, y eso tumba el sistema de métricas o dispara la factura. Esos identificadores van en logs y trazas, que están diseñados para el detalle; nunca en etiquetas de métricas.

## Buenas prácticas avanzadas

- **Instrumenta el dominio, no solo la infraestructura.** La instrumentación automática te da HTTP y base de datos, que es el "cómo", pero no sabe que `ReservarStock` es un paso crítico ni que "pedidos perdidos por fallo de pago" es la métrica que le importa a alguien. Un panel lleno de CPU y memoria puede estar todo en verde mientras el negocio se cae. Las señales que avisan antes son casi siempre las que mapean a algo que se factura.
- **Aplica *sampling* por reglas, nunca uniforme al azar.** Guardar el 100 % de las trazas es inviable con tráfico alto, pero descartar aleatoriamente hace que pierdas justo la traza del error raro que estabas buscando. La estrategia correcta (*tail sampling*) decide **después** de completar la traza: conserva siempre las que tienen error o latencia alta, y muestrea agresivamente el tráfico sano. Cuesta más de montar y es la diferencia entre tener datos y tener los datos útiles.
- **Trata los nombres de métricas y atributos como un contrato público.** Renombrar una métrica rompe en silencio todos los paneles y alertas que dependían de ella, y nadie se entera hasta que hace falta esa alerta. Seguir las convenciones semánticas de OpenTelemetry (`http.request.method`, `http.response.status_code`) en lugar de inventar nombres propios evita además tener que traducir cuando cambies de backend.
- **Pon un presupuesto de telemetría y mídelo.** El volumen de logs crece con el tráfico y con cada `Information` nuevo que alguien añade, y la factura llega meses después sin que nadie sepa qué la disparó. Medir cuántos GB genera cada servicio, y revisarlo como se revisa cualquier otro coste, es lo que evita el recorte de urgencia que apaga justo lo que hacía falta.
- **Comprueba que la telemetría sobrevive al fallo que quieres investigar.** Si los logs se escriben en el disco del contenedor que se está reiniciando en bucle, o si el exportador de trazas apunta a un backend que cae con el mismo incidente, tendrás una ceguera total en el peor momento. La telemetría tiene que salir de la máquina cuanto antes y su ruta de salida no debería compartir destino con lo que vigila.

## Recursos didácticos

- [Play with Docker](https://labs.play-with-docker.com/) — levantar la pila Grafana + Prometheus + Loki + Tempo en contenedores y mandarle telemetría real es la forma más rápida de que los tres pilares dejen de ser abstractos. Ver el mismo `traceId` saltando de un log a una traza convence más que cualquier explicación.
- [Google SRE Book, capítulo *Monitoring Distributed Systems*](https://sre.google/sre-book/monitoring-distributed-systems/) — el texto de referencia sobre qué merece la pena vigilar y qué no, incluidas las cuatro señales doradas y por qué la mayoría de las alertas sobran. Gratis y corto.
- [Documentación de OpenTelemetry](https://opentelemetry.io/docs/what-is-opentelemetry/) — la introducción conceptual explica bien la relación entre las tres señales antes de entrar en ningún SDK.

---

*En resumen: la observabilidad es poder hacerle preguntas nuevas a un sistema en producción sin haberlas anticipado — y lo que convierte tres señales sueltas en esa capacidad es que compartan el mismo `traceId`.*
