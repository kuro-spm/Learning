# Context Engineering — Guía de conceptos

Documentación introductoria sobre context engineering: cómo diseñar y gestionar todo lo que recibe un modelo de lenguaje (LLM) en cada llamada —instrucciones, historial, documentos recuperados y herramientas— dentro de un espacio limitado.

Está escrita para perfiles backend junior-medio con experiencia en APIs y bases de datos, pero sin experiencia previa integrando modelos de lenguaje. Cada ficha explica qué es el concepto o la técnica, por qué existe, cuándo se usa y lo mínimo que necesitas saber para no perderte.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Primero la idea general y su restricción fundamental; después cómo se llena el contexto con información relevante; por último, cómo se gestiona a lo largo de conversaciones y tareas largas, y cómo optimizarlo.

### 1. La idea general y su límite fundamental

Entiende primero qué es context engineering y la restricción con la que choca todo lo demás.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Context Engineering](Context-Engineering.md) | El mapa general: qué es, en qué se diferencia del prompt engineering y por qué importa. Empieza por aquí. |
| 2 | [Ventana de Contexto](Ventana-de-Contexto.md) | La restricción con la que choca todo lo demás: cuánto cabe en una sola llamada al modelo. |

### 2. Cómo se llena el contexto con información relevante (RAG)

Las piezas que permiten que un modelo responda con datos propios y actualizados en vez de solo con lo que ya sabía.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [Embeddings y Bases de Datos Vectoriales](Embeddings-y-Bases-de-Datos-Vectoriales.md) | La pieza que permite buscar por significado en vez de por coincidencia literal. |
| 4 | [Chunking](Chunking.md) | Cómo trocear documentos largos antes de indexarlos, para recuperar solo la parte relevante. |
| 5 | [RAG](RAG.md) | Cómo se combinan búsqueda y generación para responder con datos propios y actualizados. |

### 3. Cómo se gestiona el contexto en tareas y conversaciones largas

Qué hacer cuando la conversación o la tarea dura más de lo que cabe en una sola ventana de contexto.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 6 | [Memoria de Agentes](Memoria-de-Agentes.md) | Cómo un asistente "recuerda" algo entre mensajes o entre sesiones, cuando el modelo no recuerda nada por sí mismo. |
| 7 | [Compactación de Contexto](Compactacion-de-Contexto.md) | Qué hacer cuando una conversación larga se acerca al límite de la ventana de contexto. |
| 8 | [Aislamiento de Contexto](Aislamiento-de-Contexto.md) | Cómo repartir una tarea compleja entre varios agentes sin que el ruido de unos contamine a los demás. |

### 4. Optimización

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 9 | [Prompt Caching](Prompt-Caching.md) | Cómo evitar pagar dos veces por la parte del contexto que no cambia entre llamadas. |

---

> ¿Te interesa cómo se diseñan las APIs que exponen o consumen estos modelos? Echa un vistazo a la guía de [Tipos de APIs](../../arquitectura-de-software/tipos-de-apis/README.md).
