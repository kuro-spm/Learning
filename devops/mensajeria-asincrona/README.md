# Mensajería Asíncrona — Guía de tecnologías

> 🧭 Colección añadida como recomendación proactiva, no por una necesidad concreta de ningún proyecto.

Documentación introductoria sobre mensajería asíncrona: por qué desacoplar servicios con colas y eventos, qué diferencia hay entre una cola y un topic, y qué herramientas concretas (RabbitMQ, Azure Service Bus) se usan para implementarlo.

Está escrita para perfiles backend junior-medio con experiencia en APIs REST, pero sin experiencia previa en sistemas de mensajería. Cada ficha explica qué es la tecnología o el concepto, por qué existe, cuándo se usa y lo mínimo que necesitas saber para no perderte.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Está pensado para que cada concepto apoye al siguiente.

### 1. El concepto

Empieza por entender la idea antes de mirar herramientas concretas.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Mensajería Asíncrona](Mensajeria-Asincrona.md) | La idea central: por qué desacoplar con colas y eventos, y la diferencia entre colas y topics. Empieza siempre por aquí. |

### 2. Los brokers concretos

Las plataformas que implementan esas ideas. Lee al menos una de las dos según lo que uses.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [RabbitMQ](RabbitMQ.md) | El broker autogestionado más extendido. Introduce exchanges y reglas de enrutado. |
| 3 | [Azure Service Bus](Azure-Service-Bus.md) | El equivalente gestionado en Azure. Mismas ideas, sin administrar tú el broker. |

### 3. Piezas que aparecen en cualquier sistema de mensajería

Conceptos transversales, sea cual sea el broker elegido.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 4 | [Outbox Pattern](Outbox-Pattern.md) | Cómo garantizar que un evento se publica siempre, sin quedar a medias con el dato que lo origina. |
| 5 | [Dead Letter Queues](Dead-Letter-Queues.md) | Qué pasa cuando un mensaje no se puede procesar. Aplica a cualquiera de los brokers anteriores. |

---

> ¿Quieres saber cómo se observa lo que pasa dentro de estos sistemas una vez en producción? Echa un vistazo a la guía de [Observabilidad](../observabilidad/README.md) de esta misma colección.
