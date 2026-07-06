# Flyway

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Flyway es una herramienta de migraciones de base de datos basada en ficheros SQL versionados. A diferencia de EF Core Migrations, no genera el SQL por ti a partir de clases: tú escribes cada migración como un script `.sql` normal, y Flyway se encarga de aplicarlos en orden y llevar el registro de cuáles ya se ejecutaron.

## ¿Por qué existe?

No todos los proyectos usan un ORM que genere migraciones, y muchos equipos prefieren escribir el SQL a mano para tener control total sobre exactamente qué se ejecuta contra la base de datos. Flyway nació para cubrir ese hueco: da toda la disciplina de versionado y control de aplicación de las migraciones (orden, registro de qué se ejecutó, validación de que nadie ha tocado un script ya aplicado) sin atarte a ningún lenguaje ni framework concreto. Al ser agnóstico, el mismo enfoque sirve para un backend en .NET, en Java o en Python, y para prácticamente cualquier motor de base de datos relacional.

> Si EF Core Migrations es como un asistente que redacta el `ALTER TABLE` por ti a partir de tu modelo, Flyway es la libreta de notas donde escribes tú cada `ALTER TABLE`, pero con una organización estricta de qué nota va primero y cuál ya se leyó.

## ¿Cuándo y para qué se usa?

Flyway aparece en proyectos que quieren migraciones de base de datos sin acoplarse a un ORM concreto, o en organizaciones con varios servicios en distintos lenguajes que comparten la misma disciplina de versionado de esquema. Por ejemplo, un sistema de pedidos con un backend en .NET y otro servicio de informes en Java, ambos con acceso a la misma base de datos, pueden compartir la misma carpeta de migraciones Flyway como fuente única de verdad del esquema.

## Lo mínimo que necesitas saber

**1. Las migraciones son ficheros SQL con un nombre y una convención estrictos**

```
sql/
├── V1__crear_tabla_productos.sql
├── V2__crear_tabla_pedidos.sql
└── V3__anadir_columna_stock_minimo.sql
```

El prefijo `V<número>` indica el orden; el texto tras el doble guion bajo es la descripción, y Flyway rechaza aplicar una `V2` si aún no se aplicó la `V1`.

**2. Cada fichero contiene SQL normal, sin sintaxis especial**

```sql
-- V3__anadir_columna_stock_minimo.sql
ALTER TABLE Productos ADD StockMinimo INT NOT NULL DEFAULT 0;
```

**3. Ejecutar las migraciones pendientes**

```bash
flyway -url=jdbc:sqlserver://localhost:1433;databaseName=TiendaDB \
       -user=sa -password=*** migrate
```

**4. Flyway valida que los scripts ya aplicados no hayan cambiado**

Antes de aplicar nada nuevo, calcula el checksum de cada migración ya ejecutada y lo compara con el que guardó la primera vez. Si alguien editó un script después de aplicarlo, `flyway validate` (o el propio `migrate`) falla en vez de dejar que el esquema y el historial diverjan en silencio.

```bash
flyway validate
```

**5. Migraciones repetibles, para objetos que se pueden recrear sin más**

Además de las migraciones versionadas (`V...`), existen las repetibles (`R__...`), pensadas para vistas, procedimientos almacenados o funciones: se reaplican automáticamente cada vez que cambia su contenido, sin necesidad de un número de versión nuevo.

```
sql/
└── R__vista_productos_activos.sql
```

## Lo que NO hace

- **No genera el SQL por ti** — cada migración se escribe a mano; si buscas eso, EF Core Migrations es la opción cuando ya usas ese ORM.
- **No sabe nada de tus clases C#** — no hay ninguna relación automática entre el esquema y tu modelo de objetos; esa sincronización es responsabilidad tuya.
- **No aplica rollback automático de una migración ya aplicada** (en su edición gratuita) — deshacer un cambio significa escribir una migración nueva que revierta el anterior, igual que con cualquier otra herramienta de este tipo.

## Buenas prácticas avanzadas

- **Nunca edites una migración `V` ya aplicada en un entorno compartido** — el checksum que valida Flyway existe precisamente para detectar esto. Si necesitas corregir algo, añade una migración nueva; editar la antigua rompe la validación en cualquier entorno donde ya se ejecutó.
- **Usa migraciones repetibles (`R__`) solo para objetos que se puedan recrear sin pérdida** — vistas y procedimientos encajan perfectamente; una tabla no, porque "recrearla" implicaría perder sus datos.
- **Versiona los scripts junto con el código que los necesita, en el mismo repositorio y el mismo *pull request*** — así el cambio de esquema y el cambio de código que lo usa se revisan, aprueban y despliegan juntos, evitando el clásico "el código ya espera la columna nueva, pero la migración no se ha aplicado todavía".
- **Usa `flyway info` antes de un despliegue dudoso** — muestra qué migraciones están aplicadas, cuáles pendientes y si hay alguna discrepancia de checksum, antes de que `migrate` intente hacer nada.

---

*En resumen: Flyway lleva la misma disciplina de control de versiones de EF Core Migrations, pero con SQL escrito a mano y sin depender de ningún lenguaje o ORM concreto.*
