# Embeddings y Bases de Datos Vectoriales

## ¿Qué es?

Un *embedding* es una representación numérica (un vector de cientos o miles de números) del significado de un texto, generada por un modelo, de forma que textos con significado parecido producen vectores numéricamente cercanos entre sí. Una base de datos vectorial es un almacén especializado en guardar esos vectores y encontrar, muy rápido, cuáles son los más parecidos a uno dado.

## ¿Por qué existe?

Buscar información "por palabra clave" (como un `LIKE '%palabra%'` en SQL) falla en cuanto la pregunta y el documento usan palabras distintas para decir lo mismo: "cómo cancelo mi suscripción" no encuentra un documento titulado "dar de baja el plan" aunque hablen exactamente de lo mismo. Los embeddings resuelven esto convirtiendo el *significado* en números: dos frases que significan lo mismo acaban con vectores cercanos, se parezca o no el texto literal. Las bases de datos vectoriales existen porque buscar "el vector más parecido a este" entre millones de vectores por fuerza bruta (comparar uno a uno) es demasiado lento; usan índices especializados para encontrar los más cercanos en milisegundos.

> Si conoces la búsqueda por índice en una base de datos relacional: un índice normal ayuda a encontrar filas por *igualdad* o *rango* de un valor exacto; un índice vectorial ayuda a encontrar filas por *parecido de significado*, algo que un `WHERE columna = valor` no puede expresar.

## ¿Cuándo y para qué se usa?

Es la pieza base de cualquier sistema de búsqueda semántica: encontrar los documentos de un manual más relevantes para la pregunta de un usuario, recomendar productos parecidos a uno que un cliente ya vio, agrupar tickets de soporte por tema aunque estén redactados de forma distinta. Es, además, el mecanismo de recuperación que sustenta RAG (ver [RAG](RAG.md)).

## Lo mínimo que necesitas saber

**1. Generar un embedding a partir de un texto**

```csharp
var embedding = await clienteEmbeddings.GenerarAsync("¿Cómo cancelo mi suscripción?");
// embedding es un array de, por ejemplo, 1536 números en coma flotante
```

**2. Comparar dos embeddings por su cercanía (similitud coseno)**

```csharp
double similitud = SimilitudCoseno(embeddingPregunta, embeddingDocumento);
// cercano a 1: significan casi lo mismo; cercano a 0: no tienen relación
```

**3. Guardar embeddings en una base de datos vectorial junto al texto original**

```sql
INSERT INTO documentos (id, texto, embedding)
VALUES (1, 'Para dar de baja tu plan, ve a Ajustes > Suscripción...', @vector);
```

**4. Buscar los documentos más parecidos a una pregunta**

```sql
SELECT texto
FROM documentos
ORDER BY embedding <-> @embeddingPregunta   -- distancia vectorial (p. ej. pgvector)
LIMIT 5;
```

**5. Existen varias opciones para almacenar y buscar vectores**

Desde extensiones sobre una base de datos relacional ya existente (como `pgvector` sobre PostgreSQL) hasta motores dedicados en exclusiva a esto (Pinecone, Qdrant, Weaviate, Milvus). La elección depende de si ya tienes una base de datos relacional que prefieres no duplicar, o si necesitas la escala y las prestaciones de un motor especializado.

## Lo que NO hace

- **No entiende el texto como lo entiende una persona** — el embedding captura relaciones estadísticas aprendidas durante el entrenamiento, no comprensión real; dos textos pueden salir "cercanos" por motivos que no coinciden con lo que un humano consideraría relevante.
- **No es una búsqueda exacta** — nunca dice "esto es literalmente lo que preguntabas", solo "esto es lo más parecido semánticamente que se ha encontrado"; puede devolver resultados relevantes pero no idénticos, o fallar con términos muy técnicos o específicos que el modelo de embeddings apenas conoce.
- **No sustituye a los filtros estructurados** — buscar "el pedido más parecido" no reemplaza un `WHERE clienteId = 42`; en la práctica, la búsqueda vectorial se combina con filtros exactos sobre otros campos, no los sustituye.

## Buenas prácticas avanzadas

- **Usa siempre el mismo modelo de embeddings para indexar y para buscar** — los vectores de dos modelos distintos no son comparables entre sí aunque tengan el mismo tamaño; cambiar de modelo de embeddings obliga a regenerar todo el índice desde cero, no solo los documentos nuevos.
- **No indexes un documento entero como un único vector si es largo** — un embedding de un documento de 50 páginas mezcla demasiados temas distintos en un solo vector y pierde precisión; hay que fragmentar el contenido antes de generar embeddings (ver [Chunking](Chunking.md)).
- **Combina búsqueda vectorial con búsqueda por palabra clave cuando puedas (*hybrid search*)** — la búsqueda semántica falla con términos exactos poco frecuentes (códigos de producto, nombres propios, jerga muy específica) que una búsqueda por palabra clave sí encuentra; combinar ambas suele dar mejores resultados que cualquiera de las dos por separado.
- **Vigila el coste de recalcular embeddings a gran escala** — regenerar los embeddings de todo un catálogo o una base documental no es gratis ni instantáneo; planifica cuándo y cómo se actualiza el embedding de contenido que cambia (nuevo, editado, borrado) en vez de regenerar todo cada vez.

---

*En resumen: los embeddings convierten el significado de un texto en números comparables, y una base de datos vectorial encuentra rápido cuáles son los más parecidos entre millones — es la pieza que permite buscar por significado en vez de por coincidencia literal.*
