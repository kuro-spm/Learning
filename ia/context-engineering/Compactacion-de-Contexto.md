# Compactación de Contexto

## ¿Qué es?

La compactación de contexto es la técnica de reducir el tamaño de la información que se va a enviar a un LLM —normalmente resumiendo o descartando parte de un historial largo— para que quepa dentro de la ventana de contexto disponible, sin perder la información imprescindible para seguir la tarea.

## ¿Por qué existe?

Una conversación o una tarea que se alarga (muchos turnos, muchos pasos de un agente que usa herramientas) va acumulando mensajes que, reenviados todos literales en cada llamada, acaban agotando la ventana de contexto disponible. Cortar el historial sin más (quedarse solo con los últimos N mensajes) puede perder información importante que se dijo al principio. La compactación busca un término medio: en vez de guardarlo todo literal o perderlo sin más, resume lo antiguo en una versión más corta que conserve lo esencial, y libera así espacio para que la conversación siga creciendo.

> Piensa en tomar apuntes en una reunión larga: no transcribes cada palabra dicha en la primera hora, la resumes en unas líneas con las decisiones y datos importantes, para poder seguir tomando notas del resto de la reunión sin quedarte sin páginas en la libreta.

## ¿Cuándo y para qué se usa?

En cualquier conversación o proceso agéntico de duración impredecible: un asistente que va resolviendo una tarea larga con muchos pasos intermedios, un chat de soporte que se extiende durante decenas de mensajes, un agente que ejecuta herramientas repetidamente y acumula sus resultados en el historial. Se activa normalmente cuando el contexto se acerca a un umbral del límite disponible, no en cada turno.

## Lo mínimo que necesitas saber

**1. Detectar que el contexto se acerca al límite**

```csharp
int tokensActuales = ContarTokens(historial);
if (tokensActuales > ventanaContexto * 0.8)
{
    historial = await CompactarHistorial(historial);
}
```

**2. Resumir la parte más antigua, conservando los mensajes recientes literales**

```csharp
var (antiguos, recientes) = historial.SplitAt(historial.Count - 10);

var resumen = await modelo.GenerarAsync(
    $"Resume los hechos y decisiones clave de esta parte de la conversación:\n{antiguos}");

historial = new[] { new Mensaje("system", $"Resumen de lo anterior: {resumen}") }
    .Concat(recientes)
    .ToList();
```

**3. Conservar siempre ciertos elementos sin resumir**

Las instrucciones del sistema, y a menudo decisiones o restricciones explícitas del usuario, no deberían perderse en un resumen automático; se mantienen aparte, fuera de lo que se compacta.

**4. La compactación puede repetirse varias veces a lo largo de una conversación muy larga**

Cada vez que el contexto vuelve a acercarse al límite, se compacta de nuevo, resumiendo esta vez tanto el resumen anterior como los mensajes nuevos desde entonces.

## Lo que NO hace

- **No conserva el detalle exacto de lo resumido** — un resumen, por definición, pierde matices; información concreta (un número exacto, una cita textual) puede desdibujarse o perderse si se resume sin cuidado.
- **No es instantánea ni gratis** — generar el resumen requiere otra llamada al modelo, con su propio coste y latencia; compactar en exceso o con demasiada frecuencia tiene un coste real.
- **No sustituye a una buena gestión de memoria a largo plazo** — compactar reduce el tamaño de una conversación en curso; guardar información para recuperarla en sesiones futuras es un problema distinto (ver [Memoria de Agentes](Memoria-de-Agentes.md)).

## Buenas prácticas avanzadas

- **Compacta antes de llegar al límite, no cuando ya has fallado** — esperar a que una llamada falle por exceso de tokens para reaccionar es tarde; comprobar el tamaño del contexto y compactar de forma proactiva (por ejemplo, al superar el 80% de la ventana) evita errores en producción.
- **No resumas los pasos técnicos recientes de un agente que aún está actuando** — si un agente está a mitad de una tarea con herramientas (por ejemplo, ha leído un fichero hace dos pasos y lo va a necesitar en el siguiente), resumir demasiado agresivamente esa parte reciente puede hacerle perder datos que todavía necesita para completar la tarea actual.
- **Guarda el historial completo aparte, aunque el contexto activo esté compactado** — poder auditar o depurar qué pasó realmente en una conversación es valioso incluso si el modelo ya solo trabaja con la versión resumida; no descartes el original.
- **Ajusta qué se compacta según el tipo de contenido** — una conversación de texto se resume distinto a un historial de llamadas a herramientas con resultados estructurados (JSON, tablas); tratar ambos con la misma técnica de resumen genérico suele perder más de lo necesario en el segundo caso.

---

*En resumen: la compactación resume lo antiguo de una conversación para liberar espacio en la ventana de contexto, sin perder los hechos esenciales que la tarea sigue necesitando para continuar.*
