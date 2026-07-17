# Pruebas seguras en Odoo — Guía de tecnología

Colección centrada en **cómo probar cambios sobre un Odoo real sin llevártelo por delante**: dónde probar, cómo aislar los datos de producción y cómo tener siempre marcha atrás. Es el corazón práctico de la carpeta, pensado para perfiles junior que van a trastear con un sistema que ya usan personas de verdad.

Da por sabidos los [fundamentos](../fundamentos/README.md) (qué es Odoo, módulos, modo desarrollador). Aquí el foco es la operación segura: separar entornos, duplicar y neutralizar la base de datos, y respaldar antes de cada cambio.

---

## Orden de lectura recomendado

De la idea general a la práctica concreta, y termina con el checklist que reúne todo.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Entornos: desarrollo, staging y producción](Entornos-dev-staging-produccion.md) | La idea base: separar entornos para tener un sitio donde equivocarte. |
| 2 | [Duplicar la base de datos](Duplicar-Base-de-Datos.md) | El primer paso: una copia realista de producción para probar sobre datos de verdad. |
| 3 | [Neutralizar la base de datos](Neutralizar-Base-de-Datos.md) | El paso imprescindible: apagar en la copia todo lo que puede escaparse al mundo real. |
| 4 | [Backups y restauración](Backups-y-Restauracion.md) | Tu red de seguridad: cómo volver atrás cuando algo sale mal. |
| 5 | [Pruebas seguras sobre un Odoo en producción](Pruebas-Seguras-en-Produccion.md) | La síntesis: el checklist de buenas prácticas que reúne todo lo anterior. |

---

> ¿Vas a desarrollar vistas o filtros sobre esa copia? Echa un ojo a [Búsqueda y filtros](../busqueda-y-filtros/README.md).
