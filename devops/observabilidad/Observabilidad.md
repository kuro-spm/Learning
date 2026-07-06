# Observabilidad

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

La observabilidad es la capacidad de entender qué está pasando dentro de un sistema en producción a partir de lo que ese sistema expone hacia fuera —logs, métricas y trazas—, sin necesidad de modificarlo ni de conectarte a él con un depurador.

## ¿Por qué existe?

El monitoring clásico se basa en vigilar señales que ya sabes que importan: ¿está el servidor encendido?, ¿responde el endpoint de salud?, ¿ha saltado la alerta de CPU al 90%? Funciona bien mientras los fallos se parecen a los que ya has visto antes.

El problema aparece con sistemas distribuidos: una tienda online con servicios de pedidos, pagos, inventario y envío de emails, cada uno desplegado por separado. Cuando un pedido tarda 12 segundos en confirmarse, ningún dashboard predefinido tiene la respuesta, porque nadie anticipó esa pregunta exacta. Necesitas poder explorar y reconstruir qué pasó, aunque no lo hubieras previsto de antemano.

> Piensa en el salpicadero de un coche frente al diagnóstico de un taller. El salpicadero (monitoring) te enseña unas pocas métricas fijas: velocidad, nivel de gasolina, revoluciones. El ordenador de diagnóstico del taller (observabilidad) puede responder cualquier pregunta nueva sobre el motor, porque tiene acceso a datos en crudo que nadie decidió de antemano qué forma tendrían.

## ¿Cuándo y para qué se usa?

Se vuelve imprescindible en cuanto un sistema deja de ser "una sola aplicación en un solo servidor": microservicios, colas de mensajes, múltiples instancias detrás de un balanceador. Ejemplos: encontrar por qué un pedido concreto de una tienda online se ha quedado a medias entre el servicio de pagos y el de inventario; entender por qué las peticiones de un endpoint se han vuelto lentas solo para un subconjunto de usuarios; investigar un pico de errores 500 tras un despliegue reciente sin tener que reproducirlo en local.

## Lo mínimo que necesitas saber

**1. Los tres pilares no compiten, se complementan**

Cada uno responde a una pregunta distinta sobre el mismo sistema: los logs dicen *qué* pasó, las métricas dicen *cuánto* y *con qué frecuencia*, las trazas dicen *por dónde* ha pasado una petición concreta.

**2. Logs: el detalle de un evento puntual**

```json
{"timestamp": "2026-07-06T10:15:32Z", "level": "Error", "service": "pedidos", "orderId": 87, "message": "Pago rechazado por el banco"}
```

**3. Métricas: números agregados en el tiempo**

```
http_requests_total{service="pedidos", status="500"} 42
http_request_duration_seconds{service="pedidos", quantile="0.95"} 1.8
```

**4. Trazas: el recorrido completo de una petición**

```
Trace abc123 — "Confirmar pedido #87"
├─ pedidos.CrearPedido        120 ms
├─ pagos.Cobrar                 80 ms
└─ inventario.ReservarStock    200 ms
```

**5. El hilo que conecta los tres: el identificador de correlación**

Un mismo `traceId` (o `correlationId`) viaja con la petición de servicio en servicio y se incluye en cada log y cada span. Es lo que te permite pasar de "veo un error en un log" a "veo la traza completa que lo rodea".

```csharp
using (LogContext.PushProperty("TraceId", Activity.Current?.TraceId.ToString()))
{
    logger.LogError("Pago rechazado para el pedido {OrderId}", pedido.Id);
}
```

## Lo que NO hace

- **No sustituye al monitoring clásico** — las alertas y los dashboards de salud siguen haciendo falta; la observabilidad añade la capacidad de investigar lo que esas alertas no anticiparon.
- **No es gratis** — instrumentar el código, transportar los datos y almacenarlos tiene coste de rendimiento y de dinero, sobre todo a gran volumen.
- **No interpreta nada por ti** — te da datos ricos y consultables, pero decidir qué significan sigue siendo trabajo humano.
- **No es una sola herramienta** — es una práctica que se apoya en varias piezas (un sistema de logging, uno de métricas, uno de trazas) que conviene que compartan el mismo identificador de correlación.

## Buenas prácticas avanzadas

- **Vigila la cardinalidad de las métricas antes de que te explote la factura** — añadir una etiqueta con un valor casi único (`userId`, `orderId`) a una métrica multiplica el número de series temporales que hay que guardar, a veces por millones. Esa clase de dato pertenece a los logs o a las trazas, no a una etiqueta de métrica.
- **Instrumenta manualmente lo que le importa al negocio, no solo lo automático** — la instrumentación automática (HTTP entrante/saliente, acceso a base de datos) te da el "cómo", pero no sabe que "ReservarStock" es un paso crítico del negocio. Añadir spans y métricas manuales en esos puntos es lo que distingue una observabilidad útil de una genérica.
- **Aplica sampling a las trazas con criterio, no al azar** — guardar el 100% de las trazas en un sistema con mucho tráfico es inviable en coste; descartar al azar puede hacerte perder justo la traza del error raro. Las estrategias serias muestrean menos en el tráfico "normal" y conservan siempre las trazas con error o con latencia alta.
- **Define primero qué es "sano", luego construye el dashboard** — recolectar todo sin haber decidido antes qué indicadores reflejan la salud real del servicio (SLIs) lleva a paneles con cien gráficas que nadie mira. Empieza por las preguntas que te importan y añade instrumentación para responderlas, no al revés.

---

*En resumen: la observabilidad es la capacidad de hacerle preguntas nuevas a un sistema en producción sin haberlas anticipado, apoyándote en logs, métricas y trazas correlacionados entre sí.*
