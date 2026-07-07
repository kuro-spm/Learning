# Memoria de Agentes

## ¿Qué es?

La memoria de un agente de IA es el mecanismo por el que la información relevante de una conversación o de interacciones pasadas se conserva y se vuelve a incluir en el contexto de llamadas futuras, ya que el modelo en sí no recuerda nada entre una llamada y la siguiente.

## ¿Por qué existe?

Cada llamada a un LLM es independiente: el modelo no guarda ningún estado propio de una petición a otra. Si un asistente conversacional no reenviara el historial en cada mensaje, "olvidaría" literalmente todo lo dicho antes en cuanto terminara de responder. La memoria existe para resolver ese olvido, pero además para ir un paso más allá de simplemente "reenviar todo el historial": una conversación larga o muchas interacciones a lo largo del tiempo no caben enteras en la ventana de contexto, así que hace falta decidir qué conservar, cómo resumirlo y cuándo recuperarlo.

> Si conoces la diferencia entre una variable local y una base de datos: el historial reenviado en una sola conversación es como una variable que vive mientras dura la sesión; la memoria a largo plazo (lo que el agente recuerda de ti en conversaciones distintas, días después) es como guardar ese dato en una base de datos para leerlo la próxima vez que haga falta.

## ¿Cuándo y para qué se usa?

Cualquier asistente que mantenga una conversación de más de un intercambio necesita memoria a corto plazo (el propio historial de esa sesión). Un asistente que debe recordar preferencias o hechos de un usuario entre sesiones distintas (su nombre, decisiones tomadas la semana pasada, el estado de una tarea en curso) necesita además memoria a largo plazo, guardada en algún almacén persistente y recuperada cuando sea relevante.

## Lo mínimo que necesitas saber

**1. Memoria a corto plazo: el historial de la conversación actual**

```csharp
var mensajes = new List<Mensaje>
{
    new("system", instrucciones),
    new("user", "¿Cuál es el estado de mi pedido 87?"),
    new("assistant", "Tu pedido 87 está en camino, llega mañana."),
    new("user", "¿Y si quiero cambiarlo?")   // el modelo necesita los mensajes anteriores para saber de qué pedido habla
};
```

**2. Memoria a largo plazo: hechos guardados aparte y recuperados cuando hacen falta**

```csharp
await memoriaLargoPlazo.Guardar(usuarioId, "prefiere_envio_express", "true");

// en una sesión distinta, días después
var preferencias = await memoriaLargoPlazo.Recuperar(usuarioId);
var contexto = $"Preferencias conocidas del usuario: {preferencias}";
```

**3. No toda la memoria a largo plazo se recupera siempre entera**

Igual que en RAG, cuando hay muchos hechos guardados sobre un usuario o un proyecto, se recuperan solo los relevantes para la pregunta actual (por búsqueda semántica o por reglas), no el histórico completo.

**4. La memoria puede resumirse en vez de guardarse literal**

```csharp
var resumen = await modelo.GenerarAsync($"Resume en 3 frases los hechos clave de esta conversación:\n{historialCompleto}");
await memoriaLargoPlazo.Guardar(usuarioId, "resumen_conversacion", resumen);
```

## Lo que NO hace

- **No es memoria del modelo, es memoria de la aplicación** — el modelo sigue sin recordar nada por sí mismo entre llamadas; toda la persistencia la gestiona el código que rodea al modelo, guardando y reinyectando lo necesario.
- **No es infinita ni gratuita** — cuanta más memoria se reinyecta en cada llamada, más contexto se consume (ver [Ventana de Contexto](Ventana-de-Contexto.md)) y más cara y lenta es cada petición.
- **No sustituye a una base de datos de negocio** — datos estructurados que ya viven en tu sistema (el estado real de un pedido, el saldo de una cuenta) deben seguir viviendo en tu base de datos habitual; la memoria del agente es para contexto conversacional y preferencias, no para ser la fuente de verdad del negocio.

## Buenas prácticas avanzadas

- **Resume el historial antiguo en vez de arrastrarlo literal para siempre** — una conversación larga que reenvía cada mensaje anterior sin resumir agota la ventana de contexto tarde o temprano; sustituir los mensajes antiguos por un resumen conciso conserva lo importante sin pagar el coste completo (ver [Compactación de Contexto](Compactacion-de-Contexto.md)).
- **Distingue lo que hay que recordar siempre de lo que hay que recuperar bajo demanda** — unos pocos hechos críticos (el idioma preferido, una restricción importante) merecen estar siempre presentes; el resto del historial extenso se beneficia más de recuperarse solo cuando resulta relevante, como en RAG.
- **Verifica la memoria guardada de vez en cuando, no confíes en ella ciegamente para siempre** — un hecho guardado hace meses puede haber caducado (una preferencia que cambió, un dato que ya no aplica); un agente que arrastra memoria obsoleta como si fuera cierta puede dar respuestas erróneas con total confianza.
- **Separa memoria por usuario o por conversación con cuidado** — mezclar sin querer la memoria de un usuario con la de otro es un fallo de aislamiento serio, no solo un error funcional; trátalo con el mismo cuidado que cualquier dato de un usuario distinto en tu sistema.

---

*En resumen: la memoria de un agente es la información que la aplicación guarda y reinyecta activamente en cada llamada, porque el modelo por sí mismo no recuerda nada entre una petición y la siguiente.*
