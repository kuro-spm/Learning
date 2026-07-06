# Logging Estructurado

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

El logging estructurado es la práctica de escribir los logs como **datos con campos definidos** (normalmente JSON), en lugar de frases de texto libre. Cada entrada de log es un objeto con propiedades como `nivel`, `servicio`, `orderId` o `mensaje`, no una única cadena de texto.

## ¿Por qué existe?

Un log tradicional es una frase pensada para que la lea una persona:

```
2026-07-06 10:15:32 ERROR Pago rechazado para el pedido 87, motivo: fondos insuficientes
```

Funciona mientras alguien la lee línea a línea en una terminal. El problema aparece en cuanto tienes miles de líneas por minuto repartidas entre varios servicios: para encontrar "todos los pagos rechazados del pedido 87" necesitas hacer una búsqueda de texto frágil (`grep "pedido 87"`), que se rompe si alguien cambia ligeramente la redacción del mensaje.

El logging estructurado trata cada log como si fuera una fila de una tabla en vez de una línea de texto: se puede filtrar, agregar y consultar por campo, igual que harías con `WHERE orderId = 87` en SQL en lugar de buscar la palabra "87" en un documento de texto plano.

## ¿Cuándo y para qué se usa?

En cualquier aplicación que se vaya a ejecutar en producción con más de una instancia o que se integre con un sistema centralizado de logs (Elasticsearch, Seq, Azure Monitor, CloudWatch...). Ejemplos: buscar todos los errores de un usuario concreto en una app de tareas; contar cuántas veces ha fallado el envío de emails de confirmación en una tienda online durante la última hora; correlacionar los logs de tres servicios distintos que participaron en la misma petición.

## Lo mínimo que necesitas saber

**1. Cada log es un conjunto de campos, no una frase**

```json
{
  "timestamp": "2026-07-06T10:15:32Z",
  "level": "Warning",
  "service": "envio-emails",
  "orderId": 87,
  "message": "Reintentando envío de email de confirmación"
}
```

**2. El mensaje admite plantillas con nombre, no concatenación**

En lugar de construir el texto a mano, se pasa una plantilla con huecos con nombre y los valores por separado. Esto es lo que permite que el motor de logging guarde `orderId` como campo propio, no solo como parte del texto.

```csharp
logger.LogWarning("Reintentando envío de email para el pedido {OrderId}", pedido.Id);
```

**3. Los niveles de log siguen significando lo mismo**

`Debug`, `Information`, `Warning`, `Error`, `Critical` no cambian; lo que cambia es el formato en el que se guarda cada entrada.

**4. Propiedades contextuales compartidas entre logs**

Se puede "empujar" un valor (como el `traceId` de una petición) para que aparezca automáticamente en todos los logs generados durante ese ámbito, sin repetirlo en cada línea.

```csharp
using (LogContext.PushProperty("OrderId", pedido.Id))
{
    logger.LogInformation("Pedido recibido");
    logger.LogInformation("Stock reservado");
    // ambas líneas incluyen "OrderId": 87 automáticamente
}
```

**5. El destino ya no es solo un fichero de texto**

Al ser datos estructurados, el log se puede enviar a sistemas que lo indexan y lo hacen consultable (Elasticsearch, Seq, Azure Application Insights), no solo escribirlo en un `.log`.

## Lo que NO hace

- **No es un formato de fichero concreto** — el JSON es el más habitual, pero lo importante es el concepto de campos con nombre, no una extensión de archivo.
- **No sustituye a las métricas ni a las trazas** — sigue siendo un log: cuenta lo que pasó en un momento puntual, no series numéricas ni el recorrido de una petición.
- **No mejora nada si los campos son inconsistentes** — si cada servicio llama de forma distinta al mismo concepto (`orderId` en uno, `order_id` en otro), pierdes buena parte de la ventaja de poder filtrar entre servicios.
- **No hace legibles los logs para humanos por sí solo** — leer JSON en crudo en una terminal es peor que texto plano; hace falta una herramienta que lo formatee o lo indexe para sacarle partido.

## Buenas prácticas avanzadas

- **Define un vocabulario de campos común entre servicios** — acuerda de antemano que "el pedido" siempre se llama `orderId` y "el usuario" siempre `userId` en todos los servicios. Sin esa convención, cada equipo estructura los logs a su manera y la consulta entre servicios vuelve a ser tan frágil como el texto libre.
- **No metas datos sensibles en los campos "porque son fáciles de buscar"** — la misma comodidad que hace atractivo estructurar los logs hace tentador añadir el email, la tarjeta o la contraseña como campo indexado. Eso los deja buscables y exportables por cualquiera con acceso al sistema de logs, que suele ser más gente de la que tiene acceso a la base de datos.
- **Usa plantillas de mensaje siempre, nunca interpolación de strings** — escribir `logger.LogInformation($"Pedido {id} recibido")` en vez de `logger.LogInformation("Pedido {OrderId} recibido", id)` destruye la estructura: el motor de logging ya no puede extraer `OrderId` como campo propio, solo ve un texto ya construido.
- **Decide el nivel de log pensando en quién lo va a leer en producción, no en depurar en local** — un `Information` en cada iteración de un bucle es ruido que ahoga las señales reales cuando el volumen es alto; resérvalo para eventos de negocio relevantes y usa `Debug` para el detalle fino.

---

*En resumen: el logging estructurado convierte cada log de una frase para leer en una fila de datos para consultar, lo que permite filtrar, agregar y correlacionar información entre servicios con la misma facilidad que una consulta SQL.*
