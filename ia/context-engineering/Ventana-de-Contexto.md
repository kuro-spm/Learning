# Ventana de Contexto

## ¿Qué es?

La ventana de contexto (*context window*) es la cantidad máxima de texto —medida en tokens, no en caracteres ni palabras— que un modelo de lenguaje puede "ver" a la vez en una sola llamada: instrucciones, historial, documentos y la respuesta que va a generar, todo junto.

## ¿Por qué existe?

Un modelo de lenguaje no tiene memoria persistente entre llamadas ni una base de datos interna que consulte sobre la marcha: todo lo que "sabe" para responder a una petición concreta tiene que caber en esa única llamada. La arquitectura interna de estos modelos (basada en mecanismos de atención) tiene además un coste computacional que crece con el tamaño del contexto, lo que impone un límite técnico y económico a cuánto se puede procesar de una vez. Ese límite es la ventana de contexto: todo lo que quieras que el modelo tenga en cuenta —incluida la respuesta que va a generar— tiene que entrar dentro de ese tamaño.

> Piensa en la memoria RAM de un ordenador: por rápida que sea, es finita, y todo lo que un programa necesita usar en un momento dado tiene que caber ahí; lo que no cabe hay que traerlo cuando haga falta (equivalente aquí a recuperarlo con RAG), no tenerlo todo cargado a la vez.

## ¿Cuándo y para qué se usa?

Es la primera restricción con la que choca cualquier diseño de context engineering: cuánto historial de conversación se puede mantener, cuántos documentos se pueden meter en una consulta de RAG, cuánto código se le puede pasar a un asistente de programación de una vez. Cualquier aplicación con conversaciones largas, documentos extensos o múltiples fuentes de información tiene que decidir, tarde o temprano, qué hacer cuando el contenido disponible supera la ventana.

## Lo mínimo que necesitas saber

**1. Se mide en tokens, no en caracteres ni palabras**

Un token es, aproximadamente, un fragmento de palabra (en inglés, unas 4 letras de media; en otros idiomas puede variar). "Ventana de contexto" no son 3 tokens, son bastantes más, porque el modelo trocea el texto en piezas más pequeñas que una palabra completa.

```text
"Hola, ¿cómo estás?"  →  aproximadamente 6-8 tokens, no 4 palabras
```

**2. El límite es compartido entre entrada y salida**

```text
Ventana de contexto = tokens de instrucciones + historial + documentos + pregunta + respuesta generada
```

Si el contexto de entrada ya ocupa casi todo el límite, apenas queda espacio para que el modelo genere una respuesta larga.

**3. Superar el límite no es un margen de maniobra, es un error**

```text
Error: la petición supera el número máximo de tokens permitido por el modelo.
```

No hay degradación suave: hay que recortar el contenido antes de enviarlo, no confiar en que el modelo "ya se las apañará".

**4. Modelos distintos tienen ventanas de tamaños muy distintos**

Los modelos actuales van desde ventanas de unos pocos miles de tokens hasta varios cientos de miles; el tamaño disponible es una característica más a la hora de elegir modelo para una tarea, igual que el precio o la velocidad.

## Lo que NO hace

- **No es lo mismo que la "memoria" de un asistente** — una ventana de contexto grande no significa que el modelo recuerde conversaciones pasadas; cada llamada es independiente salvo que tú vuelvas a incluir ese historial (ver [Memoria de Agentes](Memoria-de-Agentes.md)).
- **No garantiza que el modelo use bien todo lo que le cabe** — que algo entre dentro del límite no significa que el modelo le preste la misma atención a todo; el contenido al principio y al final suele pesar más que el que queda enterrado en medio.
- **No es gratis usarla al completo** — cuantos más tokens de entrada, más tiempo y más coste por llamada, exista o no margen disponible.

## Buenas prácticas avanzadas

- **No llenes la ventana solo porque hay sitio** — que quepan 100.000 tokens no significa que debas mandarlos todos; más contexto irrelevante diluye la atención del modelo sobre lo que sí importa y encarece cada llamada sin mejorar la respuesta.
- **Cuidado con el efecto "perdido en el medio"** — la información situada en mitad de un contexto muy largo tiende a pesar menos en la respuesta que la que está al principio o al final; si un dato es crítico, no lo entierres en medio de un documento larguísimo.
- **Reserva presupuesto explícito para la respuesta** — si la ventana está casi llena de entrada, el modelo puede no tener espacio suficiente para completar una respuesta larga o estructurada (como un JSON grande); calcula el hueco necesario para la salida antes de decidir cuánto meter de entrada.
- **Trata el límite como una restricción de diseño, no como una excepción a manejar en producción** — un sistema que solo descubre que se ha pasado de tokens cuando falla en producción es un sistema sin presupuesto de contexto diseñado de antemano; cuenta tokens antes de enviar, no después del error.

---

*En resumen: la ventana de contexto es la RAM del modelo para esa llamada —finita, medida en tokens y compartida entre todo lo que le das y lo que te devuelve— y diseñar bien qué entra ahí es la base de todo lo demás en context engineering.*
