# Observabilidad — Guía de tecnologías

> 🧭 Colección añadida como recomendación proactiva, no por una necesidad concreta de ningún proyecto.

Documentación introductoria sobre observabilidad: qué son sus tres pilares (logs, métricas y trazas), en qué se diferencia del monitoring clásico, y qué herramientas y estándares se usan hoy para implementarla en aplicaciones .NET.

Está escrita para perfiles backend junior-medio con experiencia en APIs y bases de datos, pero sin experiencia previa en sistemas de observabilidad. Cada ficha explica qué es la tecnología o el concepto, por qué existe, cuándo se usa y lo mínimo que necesitas saber para no perderte.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Está pensado para que cada concepto apoye al siguiente.

### 1. El concepto y sus pilares

Empieza por entender la idea antes de mirar herramientas concretas.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Observabilidad](Observabilidad.md) | La idea central: qué es, en qué se diferencia del monitoring clásico y cuáles son sus tres pilares. Empieza siempre por aquí. |
| 2 | [Logging Estructurado](Logging-Estructurado.md) | El primer pilar en detalle: tratar los logs como datos consultables en vez de texto plano. |
| 3 | [Serilog](Serilog.md) | La librería .NET más habitual para aplicar logging estructurado en la práctica. |
| 4 | [Métricas](Metricas.md) | El segundo pilar: contadores, gauges e histogramas para vigilar un sistema en el tiempo. |
| 5 | [Tracing Distribuido](Tracing-Distribuido.md) | El tercer pilar: seguir una petición a través de varios servicios. |
| 6 | [OpenTelemetry](OpenTelemetry.md) | El estándar que unifica los tres pilares y evita atarte a un proveedor concreto. Ciérralo con esto. |

---

> ¿Necesitas desacoplar servicios además de observarlos? Echa un vistazo a la guía de [Mensajería asíncrona](../mensajeria-asincrona/README.md) de esta misma colección.
