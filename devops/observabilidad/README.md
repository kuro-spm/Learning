# Observabilidad — Guía de tecnologías

Cómo entender qué pasa dentro de un sistema en producción sin conectarle un depurador: los tres pilares (logs, métricas y trazas), en qué se diferencian del monitoring clásico, y qué herramientas y estándares se usan hoy para implementarlo en aplicaciones .NET.

Está escrita para perfiles backend junior-medio con experiencia en APIs y bases de datos, pero sin experiencia previa en sistemas de observabilidad. No presupone nada: cada concepto se explica desde el problema que resuelve, con código ejecutable y la salida que produce.

Todas las fichas usan el mismo ejemplo —una tienda online con los servicios `pedidos`, `pagos`, `inventario` y `emails`, y el pedido `#4711` como caso de estudio— para que las piezas encajen entre documentos.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Cada bloque se apoya en el anterior.

### 1. El concepto

Entiende la idea y el vocabulario antes de mirar herramientas. Esta ficha define los términos que usan todas las demás.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Observabilidad](Observabilidad.md) | Los tres pilares, qué pregunta responde cada uno, el `traceId` que los une y por dónde empezar. Léela siempre primero. |

### 2. Logs: el primer pilar

El pilar más cercano al día a día, y el que más rinde por lo poco que cuesta hacerlo bien.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [Logging Estructurado](Logging-Estructurado.md) | La idea que lo cambia todo: tratar los logs como datos consultables en vez de texto. La regla de las plantillas con nombre sale de aquí. |
| 3 | [Niveles de Log](Niveles-de-Log.md) | El dial de severidad y el filtrado por categoría. Transversal a todas las librerías. |
| 4 | [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md) | El sistema de serie de .NET: la capa común sobre la que se enchufa todo lo demás. |
| 5 | [ILogger&lt;T&gt;](ILogger-T.md) | La interfaz concreta que inyectas en tus clases. Cómo se usa bien en el día a día. |
| 6 | [Serilog](Serilog.md) | La librería más extendida hoy para logging estructurado en producción. |
| 7 | [NLog](NLog.md) | La alternativa, con su configuración XML externa recargable en caliente. |
| 8 | [log4net](log4net.md) | El abuelo del logging en .NET. Lo verás en código heredado; aquí, cómo convivir con él y cómo migrarlo. |

Si solo vas a leer dos de este bloque, que sean la 2 y la 3: son las que aplican con cualquier librería.

### 3. Métricas y trazas: los otros dos pilares

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 9 | [Métricas](Metricas.md) | Contadores, gauges e histogramas; por qué la media miente y por qué un identificador nunca puede ser una etiqueta. |
| 10 | [Tracing Distribuido](Tracing-Distribuido.md) | Seguir una petición a través de varios servicios y ver en qué tramo se va el tiempo. |

### 4. El estándar que lo unifica

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 11 | [OpenTelemetry](OpenTelemetry.md) | Instrumentar las tres señales una sola vez sin atarte a un proveedor. Ciérralo con esto. |

---

## Índice completo por bloque

<details>
<summary>Ver todos los archivos</summary>

**El concepto**
- [Observabilidad](Observabilidad.md)

**Logs**
- [Logging Estructurado](Logging-Estructurado.md)
- [Niveles de Log](Niveles-de-Log.md)
- [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md)
- [ILogger&lt;T&gt;](ILogger-T.md)
- [Serilog](Serilog.md)
- [NLog](NLog.md)
- [log4net](log4net.md)

**Métricas y trazas**
- [Métricas](Metricas.md)
- [Tracing Distribuido](Tracing-Distribuido.md)

**El estándar**
- [OpenTelemetry](OpenTelemetry.md)

</details>

---

## Qué señal usar para cada pregunta

El resumen que evita el error más caro de todos, que es usar la señal equivocada.

| La pregunta | La señal | Dónde se explica |
|---|---|---|
| ¿Está fallando algo ahora mismo? | Métricas | [Métricas](Metricas.md) |
| ¿Cuántos y con qué frecuencia? | Métricas | [Métricas](Metricas.md) |
| ¿En qué tramo se va el tiempo? | Trazas | [Tracing Distribuido](Tracing-Distribuido.md) |
| ¿Qué le pasó a este caso concreto? | Logs | [Logging Estructurado](Logging-Estructurado.md) |
| ¿Por qué falló? | Logs, llegando desde la traza | Las dos anteriores |

Y el error a evitar: **contar cosas con logs**. Escribir un `Information` por petición para después contar líneas funciona hasta que el volumen se multiplica; entonces cuesta una fortuna y la consulta tarda minutos. Eso es una métrica.

## Los tres errores que se pagan caros

| Error | Consecuencia | Ficha |
|---|---|---|
| Interpolar cadenas en vez de usar plantillas | Los campos no son consultables; la estructura se pierde | [Logging Estructurado](Logging-Estructurado.md) |
| Usar un identificador como etiqueta de métrica | Explosión de cardinalidad: tumba el sistema o dispara la factura | [Métricas](Metricas.md) |
| No propagar el contexto en las colas | La traza se corta justo en el salto asíncrono | [Tracing Distribuido](Tracing-Distribuido.md) |

---

> Piezas relacionadas en otras colecciones: [Mensajería asíncrona](../mensajeria-asincrona/README.md), donde propagar el `traceId` entre servicios exige trabajo manual y sin él la traza se corta; [Despliegue en un VPS](../despliegue-en-vps/README.md), para la rotación de logs del contenedor y para no exponer al público el endpoint de métricas; y [Docker](../docker/README.md), porque en un contenedor lo correcto es escribir a la salida estándar y desentenderse del destino.
