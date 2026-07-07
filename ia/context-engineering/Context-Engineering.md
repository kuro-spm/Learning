# Context Engineering

## ¿Qué es?

Context engineering es la disciplina de diseñar y gestionar todo lo que se le da a un modelo de lenguaje (LLM) para producir la respuesta correcta en cada llamada: instrucciones, documentos relevantes, historial de conversación, herramientas disponibles y ejemplos, todo dentro de un espacio limitado (la ventana de contexto).

## ¿Por qué existe?

Al principio, sacarle partido a un LLM parecía cuestión de *prompt engineering*: encontrar las palabras exactas para una única instrucción de texto. Pero en cuanto una aplicación real necesita que el modelo conozca datos privados (el catálogo de una tienda, el historial de un ticket de soporte), recuerde lo dicho hace veinte mensajes, y sepa qué herramientas puede usar, el problema deja de ser "qué le pido" y pasa a ser "qué le doy de comer, en qué orden y con qué presupuesto de espacio". Ese presupuesto es limitado (ver [Ventana de Contexto](Ventana-de-Contexto.md)) y todo compite por él: cuanta más información irrelevante o mal organizada metas, peor calidad de respuesta y más caro y lento resulta cada llamada.

Context engineering trata ese "qué le doy" como un problema de ingeniería en sí mismo, con sus propias técnicas: qué recuperar, qué resumir, qué descartar y en qué formato presentarlo.

> Si conoces el diseño de APIs: un LLM es como una función sin estado propio entre llamadas; cada vez que la invocas tienes que pasarle en el payload absolutamente todo lo que necesita saber para responder bien, porque no recuerda nada de la llamada anterior por su cuenta.

## ¿Cuándo y para qué se usa?

En cualquier aplicación que use un LLM más allá de una pregunta suelta: un chatbot de soporte que necesita el historial del cliente y el manual de producto, un asistente de código que necesita ver los ficheros relevantes de un proyecto (no todos), un buscador que responde con datos de documentos internos en vez de con el conocimiento genérico del modelo. En todos estos casos, la calidad de la respuesta depende tanto del modelo como de lo bien construido que esté el contexto que recibe.

## Lo mínimo que necesitas saber

**1. El contexto se compone de piezas con roles distintos**

```text
[Instrucciones del sistema]  → cómo debe comportarse el modelo
[Documentos recuperados]     → información externa relevante para esta pregunta
[Historial de conversación]  → lo que se ha dicho hasta ahora
[Herramientas disponibles]   → qué acciones puede pedir ejecutar
[Pregunta del usuario]       → lo que hay que responder ahora
```

**2. No todo lo que existe cabe: hay que seleccionar**

```csharp
// Mal: meter el manual entero en cada pregunta
var contexto = manualCompleto; // agota la ventana de contexto y diluye lo relevante

// Bien: recuperar solo lo relevante para esta pregunta concreta (ver RAG)
var contexto = await buscador.RecuperarRelevantes(preguntaUsuario, top: 5);
```

**3. El orden y el formato también son parte del diseño**

Poner las instrucciones más importantes al principio o al final, estructurar los documentos con delimitadores claros (XML, Markdown) y mantener un formato consistente entre piezas ayuda al modelo a interpretar cada parte, igual que un código bien indentado es más fácil de leer que el mismo código sin formato.

**4. El presupuesto de contexto se gestiona activamente en tareas largas**

Cuando una conversación crece, mantener todo el historial literal deja de caber; hace falta resumir lo antiguo, descartar lo irrelevante o recuperar solo lo necesario en cada turno (ver [Compactación de Contexto](Compactacion-de-Contexto.md) y [Memoria de Agentes](Memoria-de-Agentes.md)).

## Lo que NO hace

- **No es una técnica de entrenamiento del modelo** — no cambia sus pesos ni le "enseña" nada de forma permanente; todo lo que aporta el contexto se olvida en cuanto termina esa llamada, salvo que se guarde aparte.
- **No sustituye a un buen modelo** — un contexto perfectamente diseñado no arregla un modelo poco capaz para la tarea, igual que unos datos limpios no arreglan un algoritmo mal elegido.
- **No es "cuanto más, mejor"** — más contexto no siempre mejora la respuesta; información irrelevante o mal organizada puede distraer al modelo tanto o más que no dársela.

## Buenas prácticas avanzadas

- **Trata el contexto como un recurso escaso, no como un cajón de sastre** — cada documento, cada mensaje del historial y cada definición de herramienta ocupa espacio y compite por la atención del modelo; antes de añadir algo "por si acaso", pregúntate si de verdad ayuda a responder la pregunta concreta de este turno.
- **Separa instrucciones estables de datos variables** — las instrucciones del sistema (que no cambian entre llamadas) y los datos de esta petición concreta (que sí cambian) se benefician de tratarse distinto: las primeras son candidatas a *prompt caching* (ver [Prompt Caching](Prompt-Caching.md)), las segundas no.
- **Mide qué parte del contexto realmente influye en la respuesta** — es habitual descubrir, revisando resultados, que buena parte de lo enviado nunca cambia nada; recortarlo reduce coste, latencia y ruido a la vez.
- **Diseña para el peor caso, no para el ejemplo feliz** — un contexto que funciona bien con una pregunta corta puede desbordarse o degradar la calidad con una conversación larga o un documento enorme; probar con los casos límite es donde se nota si el diseño aguanta.

---

*En resumen: context engineering es diseñar con intención qué le llega a un LLM en cada llamada —y qué no— porque la calidad de la respuesta depende tanto del contexto que recibe como del modelo que la genera.*
