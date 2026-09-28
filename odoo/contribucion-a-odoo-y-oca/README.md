# Contribuir un fix a Odoo y a la OCA — Guía de tecnologías

Cómo llevar la corrección de un bug encontrado en código que no es propio —del núcleo de Odoo o de un módulo de la OCA— hasta que forma parte oficial del proyecto, en vez de quedarse como un parche solo en un servidor concreto.

Pensada para quien ya sabe programar en Odoo y usar git, pero nunca ha abierto un Pull Request contra un proyecto de código abierto que no controla. Cada documento cubre el proceso completo: dónde empezar, qué reglas de calidad hay que seguir, cómo se revisa el cambio y qué pasa después de que lo acepten.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 30 | [Contribuir un fix al núcleo de Odoo](Contribuir-un-Fix-al-Nucleo-de-Odoo.md) | El caso más estricto: el fabricante revisa cada cambio y exige firmar un acuerdo de contribución. |
| 31 | [Contribuir un fix a un módulo de la OCA](Contribuir-un-Fix-a-un-Modulo-OCA.md) | El caso más abierto: revisión por mantenedores de la comunidad, con linters automáticos más exigentes que compensan la ausencia de un único fabricante. |

---

> Antes de escribir el fix, conviene saber moverse por el modelo de datos y el entorno de Odoo: ver la subcolección de [fundamentos](../fundamentos/README.md) si aún no se ha leído.
