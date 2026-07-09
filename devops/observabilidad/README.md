# Observabilidad — Guía de tecnologías

> 🧭 Colección añadida como recomendación proactiva, no por una necesidad concreta de ningún proyecto.

Documentación introductoria sobre observabilidad: qué son sus tres pilares (logs, métricas y trazas), en qué se diferencia del monitoring clásico, y qué herramientas y estándares se usan hoy para implementarla en aplicaciones .NET.

Está escrita para perfiles backend junior-medio con experiencia en APIs y bases de datos, pero sin experiencia previa en sistemas de observabilidad. Cada ficha explica qué es la tecnología o el concepto, por qué existe, cuándo se usa y lo mínimo que necesitas saber para no perderte.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Está pensado para que cada concepto apoye al siguiente.

### 1. El concepto

Empieza por entender la idea antes de mirar herramientas concretas.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Observabilidad](Observabilidad.md) | La idea central: qué es, en qué se diferencia del monitoring clásico y cuáles son sus tres pilares. Empieza siempre por aquí. |

### 2. Logs: el primer pilar

El pilar más cercano al día a día: qué es un log bien hecho y con qué herramientas se escribe en .NET.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [Logging Estructurado](Logging-Estructurado.md) | El concepto clave: tratar los logs como datos consultables en vez de texto plano. |
| 3 | [Niveles de Log](Niveles-de-Log.md) | El dial de severidad de cada mensaje y cómo se filtra por entorno. Transversal a todas las librerías. |
| 4 | [Microsoft.Extensions.Logging](Microsoft-Extensions-Logging.md) | El sistema de logging de serie de .NET: la capa común sobre la que se enchufa todo lo demás. |
| 5 | [ILogger&lt;T&gt;](ILogger-T.md) | La interfaz concreta que inyectas en tus clases para escribir logs. |
| 6 | [Serilog](Serilog.md) | La librería .NET más habitual hoy para logging estructurado en producción. |
| 7 | [NLog](NLog.md) | Alternativa veterana a Serilog, con su configuración por XML externo recargable en caliente. |
| 8 | [log4net](log4net.md) | El abuelo del logging en .NET, portado de log4j. Lo verás sobre todo en código heredado. |

### 3. Métricas y trazas: los otros dos pilares

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 9 | [Métricas](Metricas.md) | El segundo pilar: contadores, gauges e histogramas para vigilar un sistema en el tiempo. |
| 10 | [Tracing Distribuido](Tracing-Distribuido.md) | El tercer pilar: seguir una petición a través de varios servicios. |

### 4. El estándar que lo unifica

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 11 | [OpenTelemetry](OpenTelemetry.md) | El estándar que unifica los tres pilares y evita atarte a un proveedor concreto. Ciérralo con esto. |

---

> ¿Necesitas desacoplar servicios además de observarlos? Echa un vistazo a la guía de [Mensajería asíncrona](../mensajeria-asincrona/README.md) de esta misma colección.
