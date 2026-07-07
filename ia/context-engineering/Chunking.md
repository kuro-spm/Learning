**¿Qué es?** Chunking es la técnica de dividir un documento largo en fragmentos ("chunks") más pequeños antes de convertirlos en embeddings y guardarlos en una base de datos vectorial, en vez de tratar el documento completo como una única pieza.

---

## ¿Por qué existe?

Un embedding de un documento entero de muchas páginas mezcla en un único vector el significado de todos sus temas, lo que lo hace poco preciso para responder a una pregunta concreta sobre una sola parte de ese documento. Fragmentar el contenido en trozos más pequeños y coherentes hace que cada embedding represente una idea más concreta, y permite además recuperar solo la parte relevante de un documento largo, no el documento entero (ver [Ventana de Contexto](Ventana-de-Contexto.md): meter documentos completos agota enseguida el presupuesto disponible).

---

## ¿Cuándo y para qué se usa?

Siempre que se prepara contenido para un sistema de búsqueda semántica o RAG: manuales de usuario, políticas internas, documentación técnica, transcripciones largas. Cualquier fuente de texto que supere unas pocas frases se beneficia de fragmentarse antes de indexarse.

---

## Lo mínimo que necesitas saber

**1. División por tamaño fijo, con solapamiento**

```csharp
// Trozos de ~500 tokens, solapando 50 para no cortar una idea a la mitad
var chunks = Fragmentar(documento, tamanoChunk: 500, solape: 50);
```

**2. División respetando la estructura del documento**

Cortar por párrafos, secciones o encabezados (en vez de por un número fijo de caracteres) suele producir fragmentos más coherentes que un corte "a ciegas" en mitad de una frase.

```text
## Política de devoluciones     ← un chunk empieza aquí, no a mitad de esta sección
Los productos pueden devolverse en un plazo de 30 días...
```

**3. Cada chunk guarda de dónde viene**

```json
{ "texto": "...", "documentoOrigen": "manual-usuario.pdf", "seccion": "Devoluciones", "pagina": 12 }
```

Así, cuando se recupera un chunk, se puede mostrar la fuente exacta o volver al documento completo si hace falta más contexto.

---

## Lo que NO hace

- **No tiene un tamaño de fragmento universal** — el tamaño óptimo depende del tipo de contenido y de cómo se vaya a usar; hay que probarlo, no asumir un número mágico.
- **No sustituye una buena estructura del documento origen** — un documento sin títulos ni párrafos claros es difícil de fragmentar bien, se corte como se corte.

---

*En resumen: el chunking trocea documentos largos en piezas manejables y coherentes antes de indexarlos, para que la búsqueda semántica recupere justo la parte relevante en vez del documento entero.*
