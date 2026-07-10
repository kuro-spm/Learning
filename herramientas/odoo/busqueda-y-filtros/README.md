# Búsqueda y filtros en Odoo — Guía de tecnología

Colección introductoria sobre **cómo se filtran y se buscan registros en Odoo**: desde el lenguaje de dominios que hay debajo de todo, hasta el *search panel* lateral y las estrategias para crear filtros que se adaptan al contexto. Pensada para perfiles junior que empiezan a desarrollar o personalizar vistas en Odoo.

A diferencia del resto de esta carpeta (centrada en probar sin romper producción), aquí el foco es de **desarrollo**: cómo se construyen las herramientas de búsqueda que ve el usuario.

---

## Orden de lectura recomendado

Cada ficha apoya a la siguiente: primero el mecanismo común, luego dónde se usa.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Dominios](Dominios.md) | El cimiento: el lenguaje con el que Odoo expresa cualquier filtro. Sin esto, lo demás no se entiende. |
| 2 | [Vista de búsqueda](Vista-de-Busqueda.md) | Dónde se declaran los filtros, agrupaciones y campos buscables de un listado. |
| 3 | [Search Panel](Search-Panel.md) | El panel lateral de filtrado por facetas: el tema estrella y el más visual. |
| 4 | [Dominios dinámicos](Dominios-Dinamicos.md) | Filtros que se calculan según el contexto: otro campo, el usuario o una lógica en Python. |

---

> ¿Vas a probar estos cambios sobre datos reales? Repasa antes cómo hacerlo sin riesgos en la guía principal de [Odoo](../README.md).
