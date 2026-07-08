# MongoDB — Guía de tecnologías

Introducción a **MongoDB**, la base de datos NoSQL orientada a documentos, pensada para quien ya conoce el modelo relacional (tablas, filas, SQL, `JOIN`) pero no ha trabajado con una base de datos documental. Explica en qué se diferencia, cuándo conviene y cómo modelar los datos para aprovecharla.

No asume experiencia previa con NoSQL: la ficha parte de lo que ya sabes de SQL y va traduciendo cada concepto (colección, documento, `_id`, aggregation pipeline) a su equivalente relacional, con ejemplos genéricos (una tienda online, un blog, pedidos).

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [MongoDB](MongoDB.md) | Qué es una base de datos documental, cómo se traduce desde SQL, CRUD, modelado (anidar vs referenciar), índices y aggregation pipeline. |

---

> Para el otro gran uso de un almacén NoSQL —guardar datos en memoria para acelerar la aplicación— mira [Caching](../caching/README.md) y en concreto [Redis](../caching/Redis.md).
