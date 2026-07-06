# Migraciones de Esquema — Guía de tecnologías

> 🧭 Colección añadida como recomendación proactiva, no por una necesidad concreta de ningún proyecto.

Cómo versionar y aplicar cambios en el esquema de una base de datos con el mismo rigor que el control de versiones del código: por qué hace falta, las dos grandes familias de herramientas (basadas en código, como EF Core, o en SQL plano, como Flyway y DbUp) y cómo aplicar esos cambios sin parar el servicio.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Migraciones de esquema](Migraciones-de-Esquema.md) | El concepto base: por qué versionar el esquema como el código y qué problema resuelve frente a scripts SQL sueltos. |
| 2 | [EF Core Migrations](EF-Core-Migrations.md) | Migraciones generadas automáticamente a partir de tus clases C#, si ya usas Entity Framework Core como ORM. |
| 3 | [Flyway](Flyway.md) | Migraciones basadas en SQL escrito a mano, agnósticas de lenguaje y framework. |
| 4 | [DbUp](DbUp.md) | La misma idea que Flyway, pero aplicada como código C# en vez de una herramienta de línea de comandos aparte. |
| 5 | [Estrategias zero-downtime](Estrategias-Zero-Downtime.md) | Cómo aplicar cambios de esquema delicados sin detener el servicio, combinando lo anterior con una secuencia de despliegue cuidadosa. |

---

> Si buscas cómo usar Entity Framework Core como ORM en general (consultas, `DbContext`, *change tracking*), consulta su [ficha en la guía de desarrollo web](../../desarrollo-web/de-wpf-a-web/Entity-Framework-Core.md).
