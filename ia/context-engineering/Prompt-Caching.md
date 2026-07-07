**¿Qué es?** Prompt caching es una técnica que permite a un proveedor de LLM reutilizar el procesamiento ya hecho sobre la parte del contexto que no cambia entre llamadas (como las instrucciones del sistema o un documento largo de referencia), en vez de volver a procesarla desde cero en cada petición.

---

## ¿Por qué existe?

Procesar el contexto de entrada tiene un coste computacional que crece con su tamaño, y ese coste se paga en cada llamada, aunque gran parte del contexto (las instrucciones del sistema, un manual de referencia extenso) sea exactamente igual que en la llamada anterior. El prompt caching evita repetir ese trabajo: la parte del contexto que coincide byte a byte con una llamada reciente se recupera de una caché en vez de volver a procesarse, lo que reduce tanto el coste como la latencia de esa parte.

---

## ¿Cuándo y para qué se usa?

En aplicaciones que repiten, llamada tras llamada, un bloque grande y estable de contexto: un asistente con instrucciones de sistema extensas que no cambian entre turnos, un sistema que consulta el mismo documento de referencia largo en varias preguntas seguidas, una conversación donde el historial previo se reenvía sin cambios en cada turno nuevo.

---

## Lo mínimo que necesitas saber

**1. Se beneficia el contenido estable, colocado al principio del contexto**

```text
[Instrucciones del sistema — estable, cacheable]
[Documento de referencia largo — estable, cacheable]
[Historial de esta conversación — cambia poco a poco]
[Pregunta nueva del usuario — siempre distinta]
```

**2. Cualquier cambio en la parte cacheada invalida la caché desde ese punto**

```text
Llamada 1: [Instrucciones A][Documento X][Pregunta 1]  → se cachea A + X
Llamada 2: [Instrucciones A][Documento X][Pregunta 2]  → reutiliza la caché de A + X
Llamada 3: [Instrucciones B][Documento X][Pregunta 3]  → cambia el principio, ya no aprovecha nada cacheado
```

**3. Suele activarse marcando explícitamente qué parte del contexto es cacheable**

Cada proveedor tiene su propio mecanismo (un parámetro o una marca en la petición) para indicar hasta qué punto del contexto se debe intentar reutilizar de la caché.

---

## Lo que NO hace

- **No cachea la respuesta generada** — solo evita reprocesar el contexto de entrada que se repite; sigue generando una respuesta nueva cada vez.
- **No ayuda si el contexto cambia siempre desde el principio** — si lo primero que se manda ya es distinto en cada llamada (por ejemplo, un dato dinámico puesto al inicio del prompt), no hay nada estable que cachear.

---

*En resumen: el prompt caching evita pagar dos veces por procesar la parte del contexto que no cambia entre llamadas — coloca lo estable al principio y lo variable al final para aprovecharlo.*
