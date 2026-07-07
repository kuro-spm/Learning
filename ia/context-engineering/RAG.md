# RAG (Retrieval-Augmented Generation)

## ¿Qué es?

RAG es una técnica que combina la búsqueda de información (*retrieval*) con la generación de texto de un LLM: antes de responder, el sistema busca los datos relevantes en una fuente propia (documentos, base de datos) y se los añade al contexto de la pregunta, para que el modelo genere la respuesta apoyándose en esa información concreta, en vez de solo en lo que ya "sabía" de su entrenamiento.

## ¿Por qué existe?

Un modelo de lenguaje solo conoce lo que había en sus datos de entrenamiento, con una fecha de corte, y no tiene acceso a información privada de una empresa (su catálogo, sus tickets de soporte, sus políticas internas) ni a datos que cambian después de ese entrenamiento. Reentrenar el modelo cada vez que cambia un documento sería carísimo y lento. RAG evita ese problema por completo: en vez de "enseñarle" el dato al modelo de forma permanente, se lo pasa como contexto en el momento de la pregunta, recuperado justo en ese instante desde la fuente actualizada.

> Piensa en un examen con chuleta permitida frente a uno de memoria: RAG es como poder consultar el libro de texto durante el examen —siempre tienes la información actualizada a mano— en vez de depender de lo que memorizaste (y quizá ya está desactualizado) antes de entrar.

## ¿Cuándo y para qué se usa?

Cuando las respuestas deben basarse en información propia y verificable: un chatbot de soporte que responde con la documentación real de un producto, un buscador interno que responde preguntas sobre las políticas de una empresa, un asistente que debe citar la fuente exacta de un dato. Es la técnica estándar para reducir que el modelo "invente" información (*alucinaciones*) cuando existe una fuente de verdad concreta a la que puede recurrir.

## Lo mínimo que necesitas saber

**1. Indexar la fuente de información de antemano**

```csharp
foreach (var chunk in FragmentarDocumentos(documentos))
{
    var embedding = await clienteEmbeddings.GenerarAsync(chunk.Texto);
    await baseVectorial.Guardar(chunk.Texto, embedding, chunk.Metadatos);
}
```

**2. Recuperar lo relevante para la pregunta concreta**

```csharp
var embeddingPregunta = await clienteEmbeddings.GenerarAsync(preguntaUsuario);
var relevantes = await baseVectorial.BuscarSimilares(embeddingPregunta, top: 5);
```

**3. Añadir lo recuperado al contexto antes de preguntar al modelo**

```csharp
var contexto = string.Join("\n---\n", relevantes.Select(r => r.Texto));

var prompt = $"""
Responde la pregunta usando SOLO la siguiente información. Si no aparece, di que no lo sabes.

{contexto}

Pregunta: {preguntaUsuario}
""";

var respuesta = await modelo.GenerarAsync(prompt);
```

**4. El modelo genera la respuesta apoyándose en ese contexto, no en su memoria**

La instrucción de "responde solo con esta información" es clave: sin ella, el modelo puede mezclar lo recuperado con conocimiento genérico de su entrenamiento, perdiendo la garantía de que la respuesta viene de la fuente correcta.

## Lo que NO hace

- **No garantiza cero errores** — reduce mucho las invenciones frente a no dar contexto, pero el modelo puede seguir interpretando mal la información recuperada o combinarla de forma incorrecta.
- **No sustituye a una buena búsqueda** — si el paso de recuperación no encuentra el documento correcto, el modelo genera una respuesta con información irrelevante o incompleta, por muy bueno que sea el modelo; la calidad de un sistema RAG depende tanto de la búsqueda como de la generación.
- **No es gratis en latencia** — añade un paso extra (buscar) antes de generar, lo que aumenta el tiempo total de respuesta frente a preguntar directamente al modelo.

## Buenas prácticas avanzadas

- **Cita la fuente en la respuesta, no solo el contenido** — devolver de dónde sale cada dato (qué documento, qué sección) permite a quien recibe la respuesta verificarla, y facilita detectar cuándo la recuperación ha traído algo poco relevante.
- **Evalúa la recuperación por separado de la generación** — si una respuesta es mala, el problema puede estar en que no se recuperó el documento correcto (falla la búsqueda) o en que, teniéndolo, el modelo lo interpretó mal (falla la generación); medir ambos pasos por separado evita arreglar el sitio equivocado.
- **Filtra por metadatos además de por similitud semántica** — combinar la búsqueda vectorial con filtros exactos (fecha, categoría, permisos del usuario que pregunta) evita que se recupere información irrelevante o, peor, información a la que ese usuario no debería tener acceso.
- **No metas más documentos recuperados de los que caben con calidad** — recuperar demasiados fragmentos "por si acaso" satura el contexto y diluye lo realmente relevante (ver el efecto "perdido en el medio" en [Ventana de Contexto](Ventana-de-Contexto.md)); suele rendir mejor recuperar pocos y muy relevantes que muchos y mediocres.

---

*En resumen: RAG le da al modelo una chuleta actualizada justo antes de responder —buscando primero, generando después— para que la respuesta se apoye en tus datos reales en vez de solo en lo que el modelo memorizó durante su entrenamiento.*
