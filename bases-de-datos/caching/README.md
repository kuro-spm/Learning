# Caching — Guía de tecnologías

> 🧭 Colección añadida como recomendación proactiva, no por una necesidad concreta de ningún proyecto.

Cómo evitar repetir trabajo costoso guardando temporalmente sus resultados: los conceptos y patrones de caching, Redis como almacén en memoria, y cómo mantener esa caché al día sin que se convierta en una fuente de datos obsoletos.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Caching](Caching.md) | Los conceptos base: por qué cachear, qué patrones existen (cache-aside, write-through, write-behind) y cuándo no merece la pena. |
| 2 | [Redis](Redis.md) | El almacén en memoria más habitual para implementar una caché compartida entre instancias. |
| 3 | [Estrategias de invalidación](Estrategias-de-Invalidacion.md) | El problema más difícil del caching: decidir cuándo un dato cacheado deja de ser válido. |
| 4 | [Caché distribuida vs. local](Cache-Distribuida-vs-Local.md) | Dónde vive la caché y cómo elegir entre las dos opciones según cómo se despliega tu aplicación. |

---

> Si buscas cómo persistir datos de forma definitiva en vez de temporal, echa un vistazo a [Acceso a datos en .NET](../acceso-a-datos-dotnet/README.md).
