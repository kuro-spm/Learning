# Migraciones de esquema — Guía de tecnologías

Cómo versionar y aplicar cambios en el esquema de una base de datos con el mismo rigor con el que se versiona el código, y cómo aplicarlos en producción sin que el despliegue devuelva errores mientras dura.

La colección se lee en tres tramos: primero el concepto y el vocabulario, independientes de herramienta; luego las tres herramientas, una por familia; y por último la disciplina que hace falta cuando no puedes parar el servicio, que es un problema distinto y no lo resuelve ninguna herramienta por ti.

Todas las fichas comparten escenario: una tienda online con la base de datos `TiendaDB`, las tablas `Productos`, `Pedidos` y `Clientes`, y —en la última ficha— la API desplegada en cuatro instancias detrás de un balanceador.

---

## Orden de lectura recomendado

### 1. El concepto

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Migraciones de esquema](Migraciones-de-Esquema.md) | El vocabulario que usan las otras cuatro fichas: qué hace inmutable a una migración aplicada, qué guarda la tabla de control, numeración secuencial frente a marca de tiempo, imperativo frente a declarativo, por qué el *rollback* automático es una ilusión, y qué cambia según si el motor tiene DDL transaccional. |

### 2. Las herramientas

Las tres resuelven el mismo problema con filosofías distintas. Elige una, no las tres.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [EF Core Migrations](EF-Core-Migrations.md) | El SQL lo genera la herramienta a partir de tus clases C#. La opción natural si el proyecto ya usa EF Core. Incluye el *snapshot* del modelo, `migrations bundle` para los pipelines y el renombrado que borra datos sin avisar. |
| 3 | [Flyway](Flyway.md) | El SQL lo escribes tú, en ficheros versionados, y la herramienta es agnóstica de lenguaje. Incluye `baseline` para adoptarla en una base de datos que ya existe, los checksums y qué hacer cuando la validación falla. |
| 4 | [DbUp](DbUp.md) | La misma idea que Flyway, pero como librería .NET invocada desde tu propio código. Incluye el orden alfabético de los scripts, el recurso embebido que falla en silencio, y los tres modos de transacción. |

### 3. Aplicarlo sin parar el servicio

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 5 | [Estrategias zero-downtime](Estrategias-Zero-Downtime.md) | Esto no lo resuelve ninguna herramienta: es una disciplina. Expand/migrate/contract, la clasificación de qué cambios son seguros y qué secuencia necesita cada uno, el *backfill* por lotes, índices sin bloquear y la cola de bloqueos que tumba el servicio entero. |

---

## Qué herramienta elegir

| Si… | Elige | Por qué |
|---|---|---|
| El proyecto ya usa EF Core como ORM | [EF Core Migrations](EF-Core-Migrations.md) | No añade herramienta ni flujo de trabajo nuevos |
| Hay servicios en varios lenguajes sobre el mismo esquema | [Flyway](Flyway.md) | Es agnóstico y una sola carpeta de migraciones sirve para todos |
| Quieres SQL a mano pero sin dependencias en el servidor de destino | [DbUp](DbUp.md) | Es un paquete NuGet: el ensamblado lleva los scripts dentro |
| El esquema lo controla un DBA o otro equipo | Ninguna de las tres | Tu aplicación no debería aplicar DDL; consume el esquema que le den |
| Necesitas validación de que nadie editó una migración aplicada | [Flyway](Flyway.md) | Es la única de las tres que compara checksums de serie |

## Los errores que se pagan caros

| Síntoma | Causa habitual | Dónde se explica |
|---|---|---|
| `Invalid column name 'StockMinimo'` en pruebas y no en local | *Schema drift*: alguien aplicó un `ALTER TABLE` a mano | [Migraciones de esquema](Migraciones-de-Esquema.md) |
| `Migration checksum mismatch` | Se editó una migración ya aplicada en un entorno compartido | [Flyway](Flyway.md) |
| `Found non-empty schema(s) but no schema history table` | Falta el `baseline` al adoptar Flyway en una base de datos existente | [Flyway](Flyway.md) |
| «No hay scripts pendientes» y tampoco se aplicó nada | Los `.sql` no están marcados como *Embedded Resource*: falla en silencio | [DbUp](DbUp.md) |
| Los scripts se aplican en un orden inesperado | El orden es alfabético, no numérico: `Script10` va antes que `Script2` | [DbUp](DbUp.md) |
| Se pierden los datos de una columna «renombrada» | El renombrado se generó como `DropColumn` + `AddColumn` | [EF Core Migrations](EF-Core-Migrations.md) |
| `The model for context has pending changes` | El modelo cambió y no se generó la migración correspondiente | [EF Core Migrations](EF-Core-Migrations.md) |
| Errores 500 durante el despliegue y no después | El esquema dejó de ser compatible con el código que aún estaba vivo | [Estrategias zero-downtime](Estrategias-Zero-Downtime.md) |
| El despliegue se queda colgado y todo el servicio se para | Cola de bloqueos: un `ALTER TABLE` esperando detrás de una consulta larga | [Estrategias zero-downtime](Estrategias-Zero-Downtime.md) |
| `The transaction log for database 'TiendaDB' is full` | Un `UPDATE` masivo en una sola transacción, sin lotes | [Estrategias zero-downtime](Estrategias-Zero-Downtime.md) |

## Comprobaciones antes de aplicar una migración en producción

```sql
-- ¿Qué migraciones cree la base de datos que tiene aplicadas?
SELECT * FROM __EFMigrationsHistory ORDER BY MigrationId DESC;   -- EF Core
SELECT * FROM flyway_schema_history ORDER BY installed_rank DESC; -- Flyway
SELECT * FROM SchemaVersions ORDER BY Id DESC;                    -- DbUp
```

```bash
# ¿Qué SQL se va a ejecutar exactamente? (nunca aplicar sin haberlo leído)
dotnet ef migrations script --idempotent -o migracion.sql
flyway info
```

La regla que resume la colección: **el nombre de una migración no dice lo que hace**. Generar el script y leerlo es lo único que detecta a tiempo un `DROP COLUMN` inesperado o un cambio que va a bloquear una tabla grande.

---

> Si buscas cómo usar Entity Framework Core como ORM en general (consultas, `DbContext`, *change tracking*), su ficha está en [Acceso a datos en .NET](../acceso-a-datos-dotnet/Entity-Framework-Core.md). Y si el cambio de esquema que estás planteando es para arreglar una consulta lenta, mira antes los índices y los planes en [PostgreSQL](../postgresql/README.md): a veces la migración que hace falta es un índice, no una tabla nueva.
