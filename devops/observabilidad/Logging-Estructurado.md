# Logging Estructurado

## ¿Qué es?

El logging estructurado consiste en escribir los logs como **datos con campos con nombre** —normalmente JSON— en lugar de frases de texto libre. Cada entrada es un objeto con propiedades (`level`, `service`, `pedidoId`, `message`), no una única cadena.

## ¿Por qué existe?

Un log tradicional es una frase pensada para que la lea una persona:

```
2026-07-27 10:15:32 ERROR Pago rechazado para el pedido 4711, motivo: fondos insuficientes
```

Funciona mientras alguien la lee línea a línea en una terminal. Deja de funcionar en cuanto hay miles de líneas por minuto repartidas entre cuatro servicios. Para encontrar "todos los pagos rechazados del pedido 4711" solo queda una búsqueda de texto:

```bash
grep "pedido 4711" app.log
```

Y eso es frágil por tres motivos que aparecen todos, siempre:

- **Se rompe si alguien cambia la redacción.** Un `"Pago rechazado para el pedido 4711"` que pasa a `"No se pudo cobrar el pedido 4711"` deja fuera del `grep` medio historial.
- **Produce falsos positivos.** `grep "4711"` también encuentra el pedido 14711 y un importe de 47,11 €.
- **No se puede agregar.** "¿Cuántos pagos se rechazaron por fondos insuficientes esta semana, agrupados por método de pago?" no tiene respuesta razonable con búsqueda de texto.

El logging estructurado trata cada log como una **fila de una tabla** en vez de una línea de texto. Se puede filtrar, agrupar y ordenar por campo, igual que con `WHERE pedidoId = 4711` en SQL en lugar de buscar la subcadena "4711" en un documento.

## ¿Cuándo y para qué se usa?

En cualquier aplicación que vaya a producción con más de una instancia, o que envíe sus logs a un sistema centralizado (Seq, Elasticsearch, Loki, Application Insights, CloudWatch). Es decir: prácticamente siempre.

El punto en que se vuelve imprescindible es cuando quieres correlacionar. Con logs de texto de cuatro servicios distintos, reconstruir qué le pasó a una petición concreta consiste en cruzar marcas de tiempo a mano. Con logs estructurados y un `traceId` común, es una consulta.

---

## La regla de oro: plantillas con nombre

Esta es la idea de la que sale todo lo demás, y también el error más común. Compara las dos formas de escribir el mismo log:

```csharp
// ❌ Interpolación: el texto llega ya construido
logger.LogWarning($"Reintentando envío de email para el pedido {pedido.Id}");

// ✅ Plantilla con nombre: el valor viaja por separado
logger.LogWarning("Reintentando envío de email para el pedido {PedidoId}", pedido.Id);
```

En consola las dos imprimen exactamente lo mismo, y por eso la primera pasa las revisiones sin que nadie diga nada. Pero lo que llega al sistema de logs es muy distinto.

Con interpolación:

```json
{
  "level": "Warning",
  "message": "Reintentando envío de email para el pedido 4711"
}
```

Con plantilla:

```json
{
  "level": "Warning",
  "message": "Reintentando envío de email para el pedido 4711",
  "messageTemplate": "Reintentando envío de email para el pedido {PedidoId}",
  "pedidoId": 4711
}
```

Las dos diferencias son las que importan:

- **`pedidoId` es un campo propio.** Se puede filtrar por él (`pedidoId = 4711`), agrupar por él y usarlo en un panel. En la versión interpolada, el 4711 solo existe dentro de una cadena.
- **`messageTemplate` es estable.** Todos los reintentos de todos los pedidos comparten la misma plantilla, así que se pueden contar como un mismo tipo de evento aunque el texto final difiera. Con interpolación, cada pedido genera un mensaje literalmente distinto y agruparlos es imposible.

Los nombres de los huecos son **posicionales, no se emparejan por nombre**: el primer `{...}` recibe el primer argumento. Esto compila y produce datos mal etiquetados sin avisar:

```csharp
// ⚠️ Los valores están cruzados: PedidoId valdrá "tarjeta"
logger.LogInformation("Pedido {PedidoId} pagado con {MetodoPago}", metodoPago, pedido.Id);
```

Por convención los nombres van en `PascalCase` dentro de la plantilla. Serilog los guarda tal cual; Microsoft.Extensions.Logging también. Elige una convención y mantenla, porque `PedidoId` y `pedidoId` son dos campos distintos para el motor de búsqueda.

## Registrar objetos completos

A veces quieres el objeto entero, no un campo. Pasarlo directamente no funciona como esperarías:

```csharp
logger.LogInformation("Pedido recibido: {Pedido}", pedido);
```

Por defecto, el motor llama a `ToString()` sobre el objeto, y salvo que sea un `record` obtienes el nombre del tipo:

```
Pedido recibido: MiTienda.Dominio.Pedido
```

En Serilog, el operador `@` fuerza a **destructurar**: serializar las propiedades del objeto como estructura anidada en lugar de convertirlo a texto.

```csharp
logger.LogInformation("Pedido recibido: {@Pedido}", pedido);
```

```json
{
  "message": "Pedido recibido: Pedido { Id: 4711, Total: 149.90, Estado: \"Confirmado\" }",
  "pedido": { "Id": 4711, "Total": 149.90, "Estado": "Confirmado" }
}
```

Ahora se puede consultar `pedido.Estado = 'Confirmado'`. Muy potente y con dos trampas serias:

- **Arrastra todo el objeto**, incluidas propiedades que quizá no quieres en un log (el email del cliente, la dirección, el número de tarjeta). Volveremos a esto.
- **Puede ser enorme.** Destructurar una entidad con colecciones cargadas genera un log de megabytes que dispara el coste y ralentiza el índice.

La regla práctica: destructura DTOs pequeños que hayas diseñado para eso, nunca entidades de dominio completas.

## Contexto compartido: los ámbitos

Repetir `pedidoId` en cada llamada dentro de una operación es tedioso y se olvida justo en el log que hacía falta. Los **ámbitos** (*scopes*) adjuntan propiedades a todo lo que se registre dentro de un bloque.

Con `Microsoft.Extensions.Logging`:

```csharp
using (logger.BeginScope(new Dictionary<string, object> { ["PedidoId"] = pedido.Id }))
{
    logger.LogInformation("Stock reservado");
    logger.LogInformation("Pago cobrado");
    logger.LogInformation("Email encolado");
    // las tres líneas llevan PedidoId = 4711 sin repetirlo
}
```

Con Serilog, el equivalente es `LogContext.PushProperty`:

```csharp
using (LogContext.PushProperty("PedidoId", pedido.Id))
{
    logger.LogInformation("Stock reservado");
    logger.LogInformation("Pago cobrado");
}
```

En ambos casos la propiedad **desaparece al salir del bloque**, y el ámbito sigue el flujo asíncrono: si dentro llamas a un método `async`, sus logs también la llevan.

Este mecanismo es lo que hace posible la correlación de la que depende toda la [observabilidad](Observabilidad.md). Un middleware que abre un ámbito con el `traceId` al principio de cada petición consigue que **todos** los logs de esa petición lo lleven, sin tocar una línea del código de negocio:

```csharp
app.Use(async (context, next) =>
{
    using (LogContext.PushProperty("TraceId", Activity.Current?.TraceId.ToString()))
    {
        await next();
    }
});
```

## Un vocabulario común entre servicios

El logging estructurado no aporta gran cosa si cada servicio nombra los campos a su manera. Con esto:

| Servicio | Cómo llama al pedido |
|---|---|
| pedidos | `pedidoId` |
| pagos | `order_id` |
| inventario | `OrderNumber` |
| emails | `id` |

...la consulta que cruza los cuatro servicios vuelve a ser tan frágil como el texto libre, porque hay que conocer y escribir las cuatro variantes.

Merece la pena acordar de antemano un puñado de nombres y respetarlos en todos los servicios:

```
traceId      el identificador de correlación (obligatorio en todos)
service      qué servicio generó el log
environment  production / staging
pedidoId     el pedido, siempre con este nombre
usuarioId    el usuario, nunca su email
```

Y donde exista una convención estándar, úsala en lugar de inventar: OpenTelemetry define nombres para lo habitual (`http.request.method`, `http.response.status_code`, `server.address`), lo que además evita traducir si algún día cambias de backend.

## Qué no debe entrar nunca en un log

La misma comodidad que hace atractivo estructurar los logs hace tentador meter en ellos cualquier dato "por si acaso". Conviene tener presente quién puede leerlos: el sistema de logs suele estar abierto a **mucha más gente** que la base de datos —desarrollo, soporte, operaciones— y sus copias de seguridad viven más tiempo.

Nunca como campo de log:

- Contraseñas, tokens, claves de API, cookies de sesión.
- Datos personales: email, teléfono, DNI, dirección.
- Datos de pago: número de tarjeta, CVV, IBAN.
- El cuerpo completo de peticiones que puedan contener cualquiera de los anteriores.

```csharp
// ❌ Deja el email indexado y buscable para siempre
logger.LogInformation("Login de {Email}", usuario.Email);

// ✅ Un identificador interno correlaciona igual, sin exponer el dato
logger.LogInformation("Login de {UsuarioId}", usuario.Id);
```

El caso que se escapa más a menudo es el destructurado: `logger.LogInformation("Usuario: {@Usuario}", usuario)` publica **todas** las propiedades, incluidas las que alguien añada al modelo dentro de seis meses sin pensar en los logs. Es un fallo que se introduce solo, sin tocar la línea del log.

Serilog permite marcar propiedades para que nunca se serialicen, lo que convierte la protección en algo estructural en vez de depender de la disciplina:

```csharp
public class Usuario
{
    public int Id { get; set; }

    [NotLogged]                       // con Destructurama.Attributed
    public string Email { get; set; }
}
```

El tema completo —por qué los secretos no deben viajar y qué hacer si se filtran— está en [gestión de secretos](../../seguridad/gestion-de-secretos-en-desarrollo/README.md).

## Cómo se consulta al final

Todo esto tiene sentido por lo que permite hacer después. Con los logs indexados en Seq, la investigación de un incidente es una consulta:

```sql
-- Todos los eventos de un pedido concreto, en orden
PedidoId = 4711
```

```sql
-- Los pagos rechazados de la última hora, agrupados por motivo
Level = 'Error' and Service = 'pagos'
| summarize count() by Motivo
```

```sql
-- Peticiones lentas de un usuario, sin conocer el texto exacto del mensaje
UsuarioId = 913 and DuracionMs > 1000
```

Esa última es la que resume la ventaja: **no hace falta saber cómo está redactado el mensaje**. En logs de texto plano, cada consulta empieza por adivinar qué palabras escribió quien programó esa línea.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| El campo no aparece en el sistema de logs | Se usó interpolación (`$"..."`) en vez de plantilla |
| El campo sale como `MiApp.Dominio.Pedido` | Falta el operador `@` para destructurar |
| Los valores están cruzados entre campos | Los argumentos se pasaron en otro orden que los huecos |
| El mismo dato aparece con dos nombres | Falta un vocabulario común entre servicios |
| Los logs pesan una barbaridad | Se está destructurando una entidad completa |
| En local no se lee nada | Se está escribiendo JSON en consola; usa un formateador legible en desarrollo |

Ese último merece una nota: JSON en crudo en una terminal es **menos** legible que el texto plano. La configuración habitual es formato legible para personas en desarrollo y JSON en producción, que es donde lo consume una máquina:

```csharp
if (builder.Environment.IsDevelopment())
    config.WriteTo.Console();                             // legible
else
    config.WriteTo.Console(new CompactJsonFormatter());   // para el agregador
```

## Buenas prácticas avanzadas

- **Analiza la coherencia de las plantillas de forma automática.** Añadir `SerilogAnalyzer` o las reglas de análisis de .NET al proyecto detecta en tiempo de compilación la interpolación, los argumentos descuadrados y los nombres de propiedad duplicados con distinta capitalización. Es la única forma de que la disciplina no dependa de que alguien lo vea en la revisión, y estos fallos son invisibles en ejecución.
- **Mantén estable la plantilla del mensaje aunque cambies el texto.** Los paneles y alertas que agrupan por `messageTemplate` se rompen en silencio si alguien reformula el mensaje. Si necesitas cambiar la redacción, ten presente que estás rompiendo un contrato igual que si renombraras una métrica; si necesitas agrupar de forma estable, un campo `EventId` explícito es más robusto que el texto.
- **No registres lo mismo en dos capas.** Es habitual que el repositorio registre "fallo al guardar", el servicio registre "no se pudo confirmar el pedido" y el middleware global registre la excepción entera. Tres entradas para un único fallo triplican el coste y hacen creer que hubo tres problemas. La convención sana: se registra donde se **maneja** la excepción, no donde se propaga.
- **Añade la duración como campo numérico, no dentro del texto.** `logger.LogInformation("Pedido procesado en {DuracionMs} ms", sw.ElapsedMilliseconds)` permite filtrar por `DuracionMs > 1000` y sacar percentiles sobre los propios logs. Escribirlo dentro de la frase lo convierte en texto inútil para cualquier análisis, y es el dato que más se acaba necesitando.
- **Pon un límite de tamaño por evento.** Un log con un cuerpo de petición grande o una colección destructurada puede ocupar megabytes; unos pocos por segundo saturan el agregador y, en algunos sistemas, provocan que se descarten eventos **incluidos los importantes**. Truncar los campos de texto largos en el momento de registrarlos es más seguro que confiar en los límites del destino.

## Recursos didácticos

- [Seq](https://datalust.co/seq) — se levanta con un contenedor (`docker run -p 5341:80 datalust/seq`) y da una interfaz donde consultar logs estructurados por campo. Mandarle unos cuantos logs propios y filtrar por `PedidoId` es lo que hace que el concepto encaje de golpe; su versión gratuita basta para desarrollo.
- [Documentación de Serilog sobre *structured data*](https://github.com/serilog/serilog/wiki/Structured-Data) — la explicación canónica de plantillas, destructuring y el operador `@`, con los casos límite bien cubiertos.
- [Convenciones semánticas de OpenTelemetry](https://opentelemetry.io/docs/specs/semconv/) — el catálogo de nombres de atributo estándar, útil para no inventar nombres cuando ya existe uno acordado.

---

*En resumen: el logging estructurado convierte cada log de una frase que leer en una fila que consultar — y todo depende de una sola disciplina, usar plantillas con nombre en lugar de interpolar cadenas.*
