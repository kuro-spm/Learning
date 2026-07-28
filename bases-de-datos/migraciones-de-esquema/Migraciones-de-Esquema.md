# Migraciones de Esquema

## ¿Qué es?

Una migración de esquema es un cambio en la estructura de una base de datos —tablas, columnas, índices, restricciones— guardado como un fichero versionado en el repositorio, revisado como cualquier otro código y aplicado en un orden fijo en todos los entornos.

## ¿Por qué existe?

El código de una aplicación vive en control de versiones: cualquiera puede ver su historial, revertir un cambio o saber exactamente qué versión corre en producción. El esquema de la base de datos, durante mucho tiempo, no ha tenido esa disciplina. Lo habitual era un `ALTER TABLE` ejecutado a mano contra cada entorno, sin registro de quién lo lanzó, cuándo, ni dónde.

Ese enfoque falla siempre por el mismo sitio: alguien aplica un cambio en su base de datos local y olvida compartirlo; nadie sabe si un script ya se ejecutó en pruebas, así que se ejecuta dos veces o ninguna; y los entornos acaban con esquemas *parecidos pero distintos* sin que nadie lo note hasta que algo revienta. Las herramientas de migración llevan al esquema el mismo rigor del control de versiones: cada cambio es un fichero con un orden explícito, y la propia herramienta anota en la base de datos qué migraciones ya se aplicaron ahí.

> Piensa en las migraciones como los *commits* de Git, pero para el esquema: cada una es un cambio pequeño con historial, y aplicarlas en orden te lleva de un esquema a otro de forma predecible.

## ¿Cuándo y para qué se usa?

En cuanto una base de datos relacional sobrevive a su primer despliegue. En esta guía el ejemplo conductor es una tienda online: la base de datos `TiendaDB` con las tablas `Productos`, `Pedidos` y `Clientes`, y un pedido concreto, el **#4711**, que iremos siguiendo. Esta ficha explica el **concepto**, independiente de herramienta: el vocabulario que después usan [EF Core Migrations](EF-Core-Migrations.md), [Flyway](Flyway.md) y [DbUp](DbUp.md). Aquí no hay comandos de ninguna de las tres.

## El *schema drift*: el día en que un equipo descubre que necesita migraciones

*Schema drift* (deriva de esquema) es que dos entornos que deberían tener la misma estructura ya no la tienen. No se anuncia: se descubre.

El síntoma llega desde el entorno de pruebas, y siempre desconcierta porque en local funciona:

```
Microsoft.Data.SqlClient.SqlException (0x80131904):
Invalid column name 'StockMinimo'.
```

La entidad `Producto` en C# tiene una propiedad `StockMinimo`, la consulta la pide, y en la base de datos de pruebas esa columna no existe: el `ALTER TABLE` se ejecutó a mano en local hace tres semanas y nunca salió de ahí.

Cuando aparece este error, lo primero es medir el daño. Esta consulta compara la estructura de dos bases de datos columna a columna y devuelve **solo las diferencias**:

```sql
SELECT COALESCE(l.TABLE_NAME, p.TABLE_NAME)   AS tabla,
       COALESCE(l.COLUMN_NAME, p.COLUMN_NAME) AS columna,
       l.DATA_TYPE AS en_local,
       p.DATA_TYPE AS en_pruebas
FROM LOCALDEV.TiendaDB.INFORMATION_SCHEMA.COLUMNS AS l
FULL OUTER JOIN TiendaDB.INFORMATION_SCHEMA.COLUMNS AS p
    ON l.TABLE_NAME = p.TABLE_NAME AND l.COLUMN_NAME = p.COLUMN_NAME
WHERE l.COLUMN_NAME IS NULL
   OR p.COLUMN_NAME IS NULL
   OR l.DATA_TYPE <> p.DATA_TYPE;
```

El `FULL OUTER JOIN` conserva las columnas que están solo en un lado, y el `WHERE` descarta todo lo que coincide. La salida es la radiografía de la deriva:

```
tabla      | columna         | en_local | en_pruebas
-----------+-----------------+----------+-----------
Productos  | StockMinimo     | int      | NULL
Pedidos    | CodigoSegimiento| nvarchar | NULL
Clientes   | Telefono        | NULL     | nvarchar
Pedidos    | Total           | decimal  | float
```

Cuatro derivas distintas y ninguna registrada en ningún sitio: dos columnas que solo existen en local, una que alguien añadió directamente en pruebas, y una que además tiene **tipos distintos** en cada entorno —`decimal` en local y `float` en pruebas— con lo que eso implica para los importes. Si `LOCALDEV` no está accesible como *linked server*, se ejecuta la mitad de la consulta en cada entorno y se comparan los dos resultados; el diagnóstico es el mismo. Ese es el momento en que un equipo adopta migraciones — y conviene dejar la consulta a mano, porque sirve igual de bien para verificar, tras un despliegue, que pruebas y producción siguen siendo la misma cosa.

## Anatomía de una migración

Toda herramienta de migración, escriba SQL o lo genere, maneja las mismas cuatro piezas:

| Pieza | Qué es | Ejemplo |
|---|---|---|
| **Identificador de orden** | Decide la posición en la secuencia | `12`, `20260203141522` |
| **Descripción** | Para qué es, legible por humanos | `anadir_columna_stock_minimo` |
| **Contenido** | El cambio en sí | `ALTER TABLE Productos ADD ...` |
| **Checksum** | Huella del contenido, para detectar ediciones | `938104772` |

En una herramienta basada en ficheros SQL, las tres primeras van en el nombre y el cuerpo del fichero:

```sql
-- V12__anadir_columna_stock_minimo.sql
ALTER TABLE Productos ADD StockMinimo INT NOT NULL DEFAULT 0;
```

El identificador es `12`, la descripción `anadir columna stock minimo` y el contenido esa línea. El **checksum** no lo escribe nadie: la herramienta lo calcula a partir del contenido del fichero la primera vez que lo aplica y lo guarda.

Ese checksum es lo que hace **inmutable** a una migración ya aplicada. En cada arranque, la herramienta recalcula la huella de los ficheros y la compara con la guardada. Si alguien edita `V12` después de que se haya aplicado en pruebas, las huellas dejan de coincidir y la herramienta se niega a seguir — porque no tiene forma de saber si la base de datos tiene lo viejo o lo nuevo. La regla práctica: **una migración que ha tocado un entorno compartido no se edita jamás; se corrige con otra migración encima**.

## La tabla de control

La herramienta necesita saber qué se aplicó ya en *esta* base de datos concreta, y lo guarda en una tabla que crea ella misma. Cambia el nombre y el detalle, no la idea:

| Herramienta | Tabla | Columnas clave |
|---|---|---|
| EF Core | `__EFMigrationsHistory` | `MigrationId`, `ProductVersion` |
| Flyway | `flyway_schema_history` | `version`, `description`, `checksum`, `success`, `execution_time` |
| DbUp | `SchemaVersions` | `Id`, `ScriptName`, `Applied` |

La de Flyway es la más informativa, así que sirve bien de ejemplo:

```sql
SELECT installed_rank, version, description, checksum, success
FROM flyway_schema_history
ORDER BY installed_rank;
```

```
installed_rank | version | description                | checksum   | success
---------------+---------+----------------------------+------------+--------
            11 | 11      | crear tabla Productos      | 1874421099 |       1
            12 | 12      | anadir columna StockMinimo |  938104772 |       1
            13 | 13      | indice Pedidos ClienteId   | -114552038 |       0
```

Se lee de arriba abajo: `installed_rank` es el orden real de aplicación, `version` el identificador declarado, y `success` dice si terminó. Ese `success = 0` de la línea 13 significa que la migración falló a mitad y **la base de datos está en un estado que no corresponde a ninguna versión**; hasta que alguien lo resuelva, la herramienta se niega a aplicar nada más.

Esta tabla es la única fuente de verdad sobre el estado del esquema, y por eso conviene entender qué pasa si se manipula:

- **Si alguien la borra**, la herramienta concluye que la base de datos está virgen e intenta aplicar `V1` desde el principio. Resultado: `There is already an object named 'Productos' in the database.` y el despliegue parado.
- **Si alguien borra una fila** para "reaplicar" una migración, esa migración se ejecuta por segunda vez sobre un esquema que ya la tiene. Un `CREATE TABLE` falla; un `ALTER TABLE ... ADD` falla; un `UPDATE` de datos, en cambio, puede ejecutarse dos veces **sin dar error** y dejar los datos mal.
- **Si alguien edita un checksum** para que cuadre con un fichero modificado, la validación pasa y el problema queda enterrado: cada entorno tiene un esquema distinto y el historial afirma que todos son iguales. No se toca a mano: las herramientas traen un comando de reparación precisamente para no tener que hacerlo.

## Numeración secuencial frente a marca de tiempo

El identificador de orden se elige de una de estas dos formas, y la decisión solo duele cuando hay ramas en paralelo.

Con **numeración secuencial** (`V13`, `V14`), dos personas que trabajan en ramas distintas crean cada una su `V13`. Al fusionar, la herramienta protesta:

```
Found more than one migration with version 13
Offenders:
-> V13__anadir_reseñas.sql
-> V13__indice_pedidos_estado.sql
```

Es molesto, pero es un conflicto **visible y en tiempo de fusión**: alguien renumera y se acabó.

Con **marca de tiempo** (`20260203141522`), no hay colisión posible: los dos identificadores son distintos por construcción. El precio es más sutil. Si la rama A se creó el lunes y se fusiona el viernes, después de la rama B del miércoles, el orden de aplicación seguirá siendo lunes → miércoles, mientras que el orden en que se revisó y probó fue miércoles → lunes. Nadie ha ejecutado nunca esa combinación antes de producción.

| | Numeración secuencial | Marca de tiempo |
|---|---|---|
| Conflictos entre ramas | Sí, y hay que renumerar | Nunca |
| Cuándo aparece el problema | Al fusionar, visible | En producción, si el orden importa |
| Orden de aplicación = orden de revisión | Sí | No necesariamente |
| Legibilidad del historial | Alta (`V13`) | Baja (`20260203141522`) |
| Encaja bien con | Equipos pequeños, una rama principal corta | Muchas ramas largas en paralelo |

Elige **secuencial** si el equipo es pequeño y las ramas viven horas: el conflicto ocasional es barato y el historial se lee. Elige **marca de tiempo** si hay varias personas con ramas de días, y compénsalo con la única regla que lo salva: **cada migración debe ser correcta sin depender de las que se fusionaron después**. En la práctica eso significa no dar por hecho el estado de otra columna que otra rama está tocando.

## Enfoque imperativo frente a declarativo

Esta distinción casi nadie la explica y es la que evita discusiones estériles entre equipos, porque los dos bandos creen estar hablando de lo mismo.

En el enfoque **imperativo** (*migration-based*) escribes los **pasos**. El repositorio contiene la secuencia de cambios; el esquema actual es el resultado de aplicarlos todos:

```sql
-- V14__renombrar_precio_a_precio_base.sql
EXEC sp_rename 'Productos.Precio', 'PrecioBase', 'COLUMN';
```

En el enfoque **declarativo** (*state-based*) escribes el **estado deseado**. El repositorio contiene la definición completa de cada tabla, y una herramienta comparadora (SSDT/DACPAC en el mundo SQL Server, o cualquier comparador de esquemas) calcula el *diff* contra la base de datos real y genera el SQL en el momento del despliegue:

```sql
-- Productos.sql — así debe quedar la tabla, sin decir cómo llegar
CREATE TABLE Productos (
    Id          INT IDENTITY PRIMARY KEY,
    Nombre      NVARCHAR(200)  NOT NULL,
    PrecioBase  DECIMAL(10,2)  NOT NULL,
    StockMinimo INT            NOT NULL DEFAULT 0
);
```

El peligro del declarativo está justo en ese ejemplo. El comparador ve una tabla con `Precio` y una definición con `PrecioBase`, y no tiene forma de saber que es un renombrado: genera `DROP COLUMN Precio` + `ADD PrecioBase`. El esquema resultante es idéntico al deseado y **todos los precios se han perdido**.

| | Imperativo (*migration-based*) | Declarativo (*state-based*) |
|---|---|---|
| Qué se revisa en el *pull request* | El paso exacto que se ejecutará | El estado objetivo; el SQL no existe aún |
| Control de los pasos intermedios | Total: tú los escribes | Ninguno: los genera el comparador |
| Riesgo de operaciones destructivas automáticas | Bajo, lo destructivo lo escribes tú | Alto: renombrados, cambios de tipo, columnas retiradas |
| Ver el estado objetivo de una tabla | Hay que reconstruirlo mentalmente | Inmediato, está en un fichero |
| Migraciones de datos y *backfill* | Naturales, son otra migración más | Encajan mal, van en scripts pre/post aparte |
| Deriva manual en la base de datos | Se detecta como fallo | Se corrige sola en silencio |

Ninguno gana siempre. El declarativo es excelente para revisar y para bases de datos con centenares de objetos, y su punto flaco se mitiga con una regla firme: **el SQL generado se inspecciona y se archiva antes de aplicarlo en producción, nunca se aplica a ciegas**. El imperativo es el que domina en aplicaciones con datos que hay que preservar, porque los pasos intermedios son exactamente el sitio donde se preservan.

## *Forward-only* frente a *rollback*

Muchas herramientas permiten escribir el cambio inverso —el `Down()` de EF Core, el `U` de Flyway— y eso genera una confianza que no está justificada. El rollback automático deshace el **esquema**; no devuelve los **datos**.

```csharp
// Up: la columna se va, y con ella su contenido
migrationBuilder.DropColumn(name: "PrecioBase", table: "Productos");

// Down: la columna vuelve... vacía
migrationBuilder.AddColumn<decimal>(
    name: "PrecioBase", table: "Productos",
    type: "decimal(10,2)", nullable: false, defaultValue: 0m);
```

Después del `Down()`, `Productos` tiene otra vez su columna `PrecioBase` y los 40 000 precios del catálogo valen `0.00`. El esquema está "revertido" y el negocio, roto. Lo mismo pasa con un `DROP TABLE` cuyo `Down()` es un `CREATE TABLE`, o con un cambio de tipo que truncó decimales.

Por eso la práctica real en producción es ***roll forward***: no se vuelve atrás, se avanza con una migración nueva que corrige.

```sql
-- V15__revertir_stock_minimo_obligatorio.sql
-- V14 puso StockMinimo NOT NULL y rompió la importación del proveedor.
-- No revertimos V14: añadimos el arreglo encima.
ALTER TABLE Productos ALTER COLUMN StockMinimo INT NULL;
```

Esto se aplica igual que cualquier otra migración, deja rastro en el historial y funciona aunque producción ya llevara horas con `V14` aplicada. Un rollback, en cambio, exigiría que ninguna otra migración se hubiera aplicado después.

Los `Down()` siguen teniendo un uso legítimo: en desarrollo local, para ir y venir entre ramas sin recrear la base de datos. Como plan de recuperación de producción, el plan es la [copia de seguridad](../../devops/despliegue-en-vps/Copias-de-Seguridad.md) más una migración correctiva, no el `Down()`.

## Migraciones de esquema frente a migraciones de datos

Un *backfill* es rellenar retroactivamente datos que antes no existían: la columna `Moneda` que acabas de añadir a `Pedidos` y que hay que poner a `'EUR'` en todo el histórico. La tentación es meterlo en la misma migración. Merece la pena calcular lo que cuesta.

```sql
-- V16__backfill_moneda.sql   ❌ dentro del despliegue
UPDATE Pedidos SET Moneda = 'EUR' WHERE Moneda IS NULL;
```

Con 12 millones de pedidos y un ritmo realista de unas 20 000 filas por segundo (una sola sentencia, un solo hilo, con los índices de la tabla actualizándose):

- **Tiempo:** 12 000 000 ÷ 20 000 ≈ **600 segundos, 10 minutos** con la tabla bloqueada y el despliegue detenido esperando.
- **Log de transacciones:** cada fila modificada registra la versión vieja y la nueva, unos 200 bytes entre ambas. 12 000 000 × 200 B ≈ **2,4 GB de log en una única transacción**, que no puede truncarse hasta que la sentencia termine. Si el fichero de log no tiene margen para crecer: `The transaction log for database 'TiendaDB' is full due to 'ACTIVE_TRANSACTION'.` La migración falla, el `ROLLBACK` de 2,4 GB tarda tanto como la propia operación, y el despliegue queda a medias.

La forma correcta es separar: la migración cambia el esquema y sale del camino crítico; el relleno va en lotes, fuera del despliegue.

```sql
-- Proceso de backfill, ejecutable por lotes y reanudable
WHILE 1 = 1
BEGIN
    UPDATE TOP (5000) Pedidos SET Moneda = 'EUR' WHERE Moneda IS NULL;
    IF @@ROWCOUNT = 0 BREAK;
    WAITFOR DELAY '00:00:00.200';   -- deja respirar al log y a las escrituras
END
```

Cada lote es una transacción corta de unos 5 000 registros: el log se trunca entre lotes, los bloqueos duran milisegundos y el proceso se puede parar y reanudar sin perder lo hecho. Tarda más en total y no le importa a nadie, porque no hay un despliegue esperando.

## DDL transaccional: lo que cambia según el motor

Si una migración tiene tres sentencias y la segunda falla, ¿se deshace la primera? Depende del motor, y esto cambia **cómo se escriben** las migraciones.

| Motor | DDL dentro de una transacción | Qué implica en la práctica |
|---|---|---|
| **PostgreSQL** | ✅ Sí, completo | Una migración fallida no deja rastro: o entera o nada |
| **SQL Server** | ✅ Sí, con matices | Casi todo el DDL es transaccional, pero el `ROLLBACK` de una operación grande puede tardar tanto como ella, y algunas sentencias (`ALTER DATABASE`, índices *full-text*) no participan |
| **MySQL / MariaDB** | ❌ No | Cada sentencia DDL hace un `COMMIT` implícito; MySQL 8 garantiza que cada una es atómica **por separado**, no el conjunto |
| **Oracle** | ❌ No | Igual: `COMMIT` implícito antes y después de cada DDL |

La consecuencia es concreta. Esta migración, en PostgreSQL, es segura tal cual:

```sql
ALTER TABLE Pedidos ADD COLUMN CodigoSeguimiento varchar(40);
ALTER TABLE Pedidos ADD COLUMN Transportista varchar(60) NOT NULL;  -- falla: tabla con filas
CREATE INDEX ix_pedidos_seguimiento ON Pedidos (CodigoSeguimiento);
```

Falla la segunda línea, se deshace la primera y la base de datos queda exactamente como estaba. En MySQL, la primera columna **ya está creada y confirmada**; la herramienta marca la migración como fallida (`success = 0`) y se detiene. Alguien tiene que entrar, mirar qué se aplicó, deshacerlo a mano y ejecutar el comando de reparación de la herramienta antes de poder desplegar otra vez.

De ahí dos reglas según el motor: en PostgreSQL puedes agrupar varios cambios relacionados en una migración; en MySQL u Oracle conviene **una migración = un cambio**, para que "quedó a medias" tenga siempre una respuesta trivial.

## Cuándo y dónde se aplican

Las migraciones se aplican como un **paso explícito del pipeline de despliegue**, antes de arrancar la versión nueva de la aplicación, con credenciales que solo se usan para eso y con la salida guardada en el registro del despliegue.

Las dos alternativas son peores:

- **A mano por SSH**, porque depende de que alguien se acuerde, y ese alguien acaba siendo la única persona que sabe en qué estado está producción.
- **Al arrancar la aplicación**, que parece cómodo y es la trampa clásica. Con una sola instancia funciona. Con cuatro réplicas arrancando a la vez tras un despliegue, las cuatro leen la tabla de control, las cuatro concluyen que falta `V17` y las cuatro ejecutan el mismo `CREATE TABLE`. Tres fallan con `There is already an object named 'Resenas' in the database.`, sus contenedores mueren, el orquestador los reinicia y el bucle se repite.

Las herramientas mitigan esto tomando un **bloqueo** antes de mirar el historial: un *advisory lock* en PostgreSQL, `sp_getapplock` en SQL Server, o una tabla dedicada (`__EFMigrationsLock`). La primera instancia lo obtiene y migra; el resto espera y, al entrar, ya no ve nada pendiente.

Que exista el bloqueo no lo convierte en buena idea, por tres motivos:

1. **El bloqueo tiene tiempo de espera.** Si la migración tarda cuatro minutos y el arranque de las réplicas expira antes, mueren igual — y a la vista del orquestador es un despliegue fallido.
2. **Un proceso que muere sin liberar el bloqueo puede dejar la fila puesta.** La siguiente instancia espera un bloqueo que ya no pertenece a nadie, y hay que borrarlo a mano.
3. **Nadie mira la migración antes de que ocurra.** Si es destructiva o tarda diez minutos, te enteras durante el arranque, con el tráfico ya llegando.

Y queda un problema que ningún bloqueo resuelve: durante el despliegue conviven código viejo y esquema nuevo, o al revés. Cómo secuenciar cada tipo de cambio para que ambas versiones funcionen a la vez es el tema de [Estrategias zero-downtime](Estrategias-Zero-Downtime.md).

## Datos de semilla frente a datos de referencia

En una migración solo entran los datos **sin los cuales el esquema no tiene sentido**: los estados posibles de un pedido, las categorías del catálogo, los países de envío. Son parte de la estructura y deben existir en todos los entornos, producción incluida.

```sql
-- V18__estados_de_pedido.sql
MERGE EstadosPedido AS destino
USING (VALUES (1,'Pendiente'), (2,'Pagado'), (3,'Enviado'), (4,'Entregado'), (5,'Devuelto'))
      AS origen (Id, Nombre)
    ON destino.Id = origen.Id
WHEN NOT MATCHED BY TARGET THEN
    INSERT (Id, Nombre) VALUES (origen.Id, origen.Nombre)
WHEN MATCHED AND destino.Nombre <> origen.Nombre THEN
    UPDATE SET Nombre = origen.Nombre;
```

El `MERGE` la hace **reejecutable sin daño**: inserta lo que falta, corrige lo que cambió y no toca lo demás. Un `INSERT` pelado fallaría con `Violation of PRIMARY KEY constraint` la segunda vez, y en un entorno donde alguien ya hubiera metido esas filas a mano rompería el despliegue.

Lo que **no** entra: el cliente Ana Ruiz, el pedido #4711 de ejemplo, los 200 productos de prueba. Eso no es esquema, es un juego de datos de un entorno concreto; va en un script de siembra aparte que solo se ejecuta en desarrollo y pruebas. Si acaba en una migración, tarde o temprano acaba en producción.

## Errores frecuentes

| Síntoma | Causa |
|---|---|
| `Invalid column name 'StockMinimo'` en pruebas, en local funciona | El cambio se aplicó a mano en local y nunca se convirtió en migración: *schema drift* |
| `Migration checksum mismatch for migration version 12` | Alguien editó una migración ya aplicada; el contenido del fichero ya no coincide con la huella guardada |
| `Found more than one migration with version 13` | Dos ramas crearon el mismo número secuencial y la fusión no renumeró ninguna |
| `Detected failed migration to version 13` y todo despliegue posterior parado | Una migración quedó a medias (`success = 0`); hay que limpiar el estado parcial y ejecutar la reparación |
| `There is already an object named 'Resenas' in the database.` | La migración se aplica sobre un objeto que ya existe: o alguien lo creó a mano, o se borró su fila del historial, o varias instancias migran a la vez |
| `Column name 'StockMinimo' in table 'Productos' is specified more than once.` | Alguien aplicó el `ALTER TABLE` a mano antes de que la migración llegara; el historial ya no describe la realidad |
| La columna renombrada existe y está toda a `0` o `NULL` | No fue un renombrado: se generó `DROP` + `ADD`. Los datos no están, y ningún error lo avisó |
| `The transaction log for database 'TiendaDB' is full` | Un `UPDATE` masivo dentro de la migración; hay que trocearlo en lotes fuera del despliegue |
| El despliegue tarda diez minutos más de lo previsto y nadie sabe por qué | La migración espera un bloqueo de esquema detrás de una consulta larga, y todo lo demás se encola detrás |
| Los datos de referencia están duplicados tras reaplicar | La migración de datos no era reejecutable (`INSERT` en vez de `MERGE`) y se ejecutó dos veces |

## Cuándo NO hacen falta migraciones

Hay tres casos donde montar todo este aparato solo añade ceremonia:

- **Bases de datos efímeras que se recrean desde cero.** Un contenedor de base de datos que arranca vacío en cada ejecución de la batería de pruebas de integración no necesita historial: crear el esquema de golpe desde su definición actual es más rápido y no acumula deuda. La única precaución es que ese esquema se genere de la **misma fuente** que produce las migraciones; si se mantiene un script paralelo a mano, tendrás deriva entre las pruebas y producción.
- **Esquemas que son propiedad de otro equipo o producto.** Si tu servicio solo lee de la base de datos de un ERP de terceros, no te toca versionar su esquema; lo que te toca es aislar la dependencia y detectar cuándo cambia. Aplicar migraciones ahí sería, además, romper el soporte del producto.
- **Almacenes sin esquema fijo.** En un almacén documental o clave-valor no hay `ALTER TABLE` que aplicar. Ojo: el problema no desaparece, se muda. El "esquema" pasa a estar implícito en el código que lee los documentos, y sigue haciendo falta versionarlo — solo que como lógica capaz de leer las formas viejas y nuevas a la vez, no como migración.

Y en ningún caso las migraciones sustituyen a la copia de seguridad: una migración mal escrita pierde datos exactamente igual que un script suelto. La única diferencia es que deja constancia de quién la escribió.

## Buenas prácticas avanzadas

- **Trata la migración aplicada como inmutable, y trata el historial como código de producción.** En cuanto una migración se ha ejecutado en un entorno que otros usan, editarla es divergir: el checksum deja de cuadrar, o peor, cuadra a la fuerza y cada entorno acaba con un esquema distinto mientras el historial afirma que son iguales. Corregir siempre con una migración nueva encima; la tabla de control no se edita a mano ni para "arreglarlo rápido".
- **Que cada migración sea reanudable tras un fallo, no reejecutable desde cero.** Lo que tiene que funcionar es reintentar un despliegue que murió a mitad, y eso depende del motor: con DDL transaccional (PostgreSQL) la herramienta ya te lo da; sin él (MySQL, Oracle) lo consigues escribiendo **un cambio por migración**, para que el estado parcial sea siempre trivial de identificar. Las migraciones de datos son la excepción: esas sí deben poder reejecutarse sin daño, con `MERGE` o con un `WHERE` que excluya lo ya hecho.
- **Revisa el SQL que la herramienta genera por ti, sobre todo si el enfoque es declarativo.** Un renombrado se convierte en `DROP` + `ADD` y se lleva los datos por delante sin dar un solo error; un cambio de tipo puede reescribir la tabla entera y bloquearla durante minutos. En declarativo, genera el script contra una copia del esquema de producción, léelo, y archívalo junto al despliegue: es la diferencia entre revisar el cambio y enterarte de él.
- **Pon `lock_timeout` (o `SET LOCK_TIMEOUT`) antes de todo DDL en producción.** Sin él, el peor escenario no es que la migración falle: es que se quede esperando un bloqueo de esquema detrás de una consulta larga, y que **toda la aplicación se encole detrás de ella**. Con un tiempo de espera de tres segundos, la migración falla rápido, el despliegue se detiene limpio y lo reintentas en un momento tranquilo, con la aplicación intacta.
- **Mide la migración contra un volumen de datos parecido al real antes de que salga.** Una migración probada solo contra las 50 filas de tu base de datos local no te dice nada sobre los 12 millones de producción. Restaura una copia reciente, aplícala, cronométrala: es la única forma de saber si el despliegue durará ocho segundos u once minutos, y hacerlo cambia decisiones de diseño, no solo de calendario.

## Recursos didácticos

- [Evolutionary Database Design, de Fowler y Sadalage](https://martinfowler.com/articles/evodb.html) — el artículo que fijó estas ideas antes de que existieran las herramientas actuales. Léelo para entender *por qué* el esquema se versiona como código; sigue explicándolo mejor que la documentación de cualquier herramienta.
- [Refactoring Databases, de Ambler y Sadalage](https://databaserefactoring.com/) — catálogo de refactorizaciones de base de datos, cada una con su secuencia de pasos y su periodo de transición. Es el sitio donde mirar cuando el cambio que necesitas no es "añadir una columna": dividir una tabla, mover una columna, sustituir una clave.
- [Strong Migrations](https://github.com/ankane/strong_migrations) — la herramienta es de Ruby, pero su README es la mejor lista que hay de qué operaciones de esquema son peligrosas, por qué bloquean, y cuál es la secuencia segura para cada una. Traducible a cualquier lenguaje.

---

*En resumen: una migración de esquema es a la base de datos lo que un commit es al código — un cambio pequeño, versionado e inmutable que convierte "qué esquema hay en cada entorno" en una pregunta con respuesta exacta; lo que no te da es marcha atrás, porque el esquema se revierte y los datos no.*
