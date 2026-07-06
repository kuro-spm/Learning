# Migraciones de Esquema

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Una migración de esquema es un cambio versionado y reproducible en la estructura de una base de datos (tablas, columnas, índices, restricciones), tratado como código: se guarda en el repositorio, se revisa y se aplica de forma ordenada en cada entorno.

## ¿Por qué existe?

El código de una aplicación vive en control de versiones: cualquiera puede ver su historial, revertir un cambio o saber exactamente qué versión corre en producción. Durante mucho tiempo, el esquema de la base de datos no ha tenido esa misma disciplina: es habitual encontrar equipos que aplican cambios sueltos con un script SQL ejecutado a mano, sin registro de quién lo lanzó, cuándo, ni en qué entornos.

Ese enfoque falla de formas muy concretas: un desarrollador aplica un `ALTER TABLE` en su base de datos local y se le olvida compartirlo; el entorno de pruebas y el de producción acaban con esquemas ligeramente distintos sin que nadie lo note hasta que algo falla; nadie sabe si un script ya se ejecutó en un entorno determinado, así que se ejecuta dos veces (o no se ejecuta ninguna). Las herramientas de migración resuelven esto llevando el mismo rigor de control de versiones al esquema: cada cambio es un fichero versionado, con un orden explícito, y la propia herramienta lleva un registro de qué migraciones ya se aplicaron en cada base de datos.

> Piensa en las migraciones como los *commits* de Git, pero para el esquema de la base de datos: cada una es un cambio pequeño y con historial, y aplicarlas en orden te lleva de un esquema a otro de forma predecible.

## ¿Cuándo y para qué se usa?

Cualquier aplicación con una base de datos relacional que evoluciona con el tiempo necesita migraciones: una tienda online que añade una tabla de reseñas de producto, un sistema de pedidos que necesita una nueva columna para el número de seguimiento del envío, o un catálogo de productos que cambia el tipo de una columna de precio. Las migraciones permiten que ese cambio se despliegue de forma coordinada en desarrollo, pruebas y producción, con la certeza de que el orden y el contenido de los cambios es el mismo en todos los entornos.

## Lo mínimo que necesitas saber

**1. Cada migración es un cambio pequeño, versionado y ordenado**

```sql
-- V12__anadir_columna_stock_minimo.sql
ALTER TABLE Productos ADD StockMinimo INT NOT NULL DEFAULT 0;
```

**2. La herramienta lleva un registro de qué se ha aplicado ya**

Las herramientas de migración crean su propia tabla de control (por ejemplo `__EFMigrationsHistory` o `flyway_schema_history`) donde anotan qué migraciones ya se ejecutaron en esa base de datos concreta, para no repetirlas ni saltarse ninguna.

```sql
SELECT * FROM flyway_schema_history;
-- version | description                 | installed_on
-- 11      | crear tabla productos       | 2026-01-10
-- 12      | anadir columna stock minimo | 2026-02-03
```

**3. Las migraciones se aplican en orden, nunca al revés sin más**

Si necesitas deshacer un cambio, la práctica habitual no es "editar" la migración ya aplicada (eso rompería el historial en cualquier entorno donde ya se ejecutó), sino escribir una migración nueva que revierta el cambio.

**4. Dos enfoques: basadas en código o basadas en SQL**

Hay herramientas que generan el SQL por ti a partir de tus clases (como EF Core Migrations) y herramientas donde tú escribes el SQL directamente en ficheros versionados (como Flyway o DbUp). Ambas resuelven el mismo problema de fondo con filosofías distintas.

**5. Se aplican como parte del despliegue, no a mano**

En un pipeline de CI/CD maduro, aplicar las migraciones pendientes es un paso automatizado más del despliegue, ejecutado contra el entorno correspondiente, no una tarea manual que alguien recuerda hacer.

## Lo que NO hace

- **No sustituye a las copias de seguridad** — una migración mal escrita puede perder datos igual que un script suelto; sigue haciendo falta backup antes de cambios importantes en producción.
- **No resuelve por sí sola el problema del *downtime*** — aplicar un cambio de esquema grande sigue pudiendo bloquear una tabla o requerir coordinación con el despliegue del código (ver [Estrategias zero-downtime](Estrategias-Zero-Downtime.md)).
- **No sabe si el cambio es semánticamente correcto** — la herramienta garantiza que el SQL se ejecuta en el orden correcto, no que la lógica del cambio sea la adecuada para tu negocio.

## Buenas prácticas avanzadas

- **Nunca edites una migración ya aplicada en algún entorno compartido** — en cuanto una migración se ejecutó en un entorno que otros usan (pruebas, producción), su contenido debe considerarse inmutable. Editarla hace que el checksum que la herramienta guardó ya no coincida, o peor, que distintos entornos acaben con historiales divergentes sin que nadie lo note.
- **Toda migración debería ser idempotente frente a reintentos, no frente a reejecución completa** — si un despliegue falla a mitad de aplicar migraciones, el reintento debe poder continuar donde quedó, no fallar por "la tabla ya existe". La mayoría de herramientas ya lo garantizan si no rompes su modelo (por ejemplo, mezclando cambios manuales con migraciones).
- **Separa cambios de esquema de cambios de datos masivos** — una migración que además hace un `UPDATE` sobre millones de filas puede bloquear el despliegue durante minutos. Si el volumen de datos es grande, conviene un proceso de backfill aparte, fuera del camino crítico del despliegue.
- **Revisa el SQL generado antes de aplicarlo en producción, aunque la herramienta lo genere por ti** — un cambio de tipo de columna o un renombrado puede traducirse en operaciones mucho más costosas o destructivas de lo que parece a simple vista (una columna "renombrada" puede en realidad crearse y borrarse, perdiendo los datos).

---

*En resumen: una migración de esquema es a la base de datos lo que un commit es al código — un cambio pequeño, versionado y reproducible que convierte "qué esquema hay en cada entorno" en una pregunta con respuesta exacta.*
