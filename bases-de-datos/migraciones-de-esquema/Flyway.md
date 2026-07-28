# Flyway

## ¿Qué es?

Flyway es una herramienta de migraciones de base de datos basada en ficheros SQL versionados: tú escribes cada cambio de esquema como un `.sql` normal y Flyway se encarga de aplicarlos en orden, una sola vez cada uno, y de anotar en la propia base de datos cuáles ya se ejecutaron.

## ¿Por qué existe?

No todos los proyectos usan un ORM que genere migraciones, y muchos equipos prefieren escribir el SQL a mano para controlar exactamente qué se ejecuta contra la base de datos. Flyway cubre ese hueco: aporta la disciplina de versionado (orden garantizado, registro de lo aplicado, validación de que nadie ha tocado un script ya ejecutado) sin atarte a ningún lenguaje ni framework. Al ser agnóstico, el mismo enfoque sirve para un backend en .NET, en Java o en Python, y para casi cualquier motor relacional.

> Si un ORM como EF Core es un asistente que redacta el `ALTER TABLE` por ti a partir de tu modelo, Flyway es la libreta de notas donde escribes tú cada `ALTER TABLE` — pero con una organización estricta de qué nota va primero y cuál ya se leyó.

Los conceptos de fondo (qué es una migración, por qué hace falta una tabla de control, imperativo frente a declarativo) están en [Migraciones de esquema](Migraciones-de-Esquema.md); esta guía los da por sabidos y va directa a cómo se usa Flyway.

## ¿Cuándo y para qué se usa?

Aparece en proyectos que quieren migraciones sin acoplarse a un ORM, y en organizaciones con varios servicios en distintos lenguajes que comparten esquema: un backend en .NET y un servicio de informes en Java que leen la misma base de datos pueden compartir una única carpeta de migraciones Flyway como fuente de verdad. Con EF Core eso no es posible, porque las migraciones viven dentro del proyecto .NET.

El ejemplo conductor de esta guía es una tienda online: base de datos `TiendaDB`, tablas `Productos`, `Pedidos` y `Clientes`, y un pedido concreto, el **#4711**, que iremos siguiendo.

## Puesta en marcha con Docker

Se puede instalar como CLI descargable o como plugin de Maven/Gradle, pero la forma más limpia de probarlo es la imagen oficial `flyway/flyway`: no instala nada en tu máquina y es exactamente la que usarás luego en el pipeline.

Primero, una carpeta `sql/` con tres ficheros —`V1__crear_tabla_productos.sql`, `V2__crear_tabla_pedidos.sql`, `V3__anadir_columna_stock_minimo.sql`— y dentro de cada uno, SQL corriente sin sintaxis especial:

```sql
-- V3__anadir_columna_stock_minimo.sql
ALTER TABLE Productos ADD StockMinimo INT NOT NULL DEFAULT 0;
```

Y ahora el contenedor, montando esa carpeta en `/flyway/sql`, que es donde Flyway busca por defecto:

```bash
docker run --rm -v "$(pwd)/sql:/flyway/sql" flyway/flyway \
  -url="jdbc:sqlserver://host.docker.internal:1433;databaseName=TiendaDB;encrypt=false" \
  -user=sa -password='Clave.Segura.2026' migrate
```

`host.docker.internal` es el truco para que el contenedor alcance un SQL Server que corre en tu portátil; si la base de datos también está en Docker, se usa el nombre del servicio y una red compartida. La salida del primer `migrate` sobre una base vacía:

```
Flyway Community Edition 10.17.0 by Redgate
Database: jdbc:sqlserver://host.docker.internal:1433;databaseName=TiendaDB (Microsoft SQL Server 16.0)
Successfully validated 3 migrations (execution time 00:00.031s)
Creating Schema History table [TiendaDB].[dbo].[flyway_schema_history] ...
Current version of schema [dbo]: << Empty Schema >>
Migrating schema [dbo] to version "1 - crear tabla productos"
Migrating schema [dbo] to version "2 - crear tabla pedidos"
Migrating schema [dbo] to version "3 - anadir columna stock minimo"
Successfully applied 3 migrations to schema [dbo], now at version v3 (execution time 00:00.184s)
```

Dos líneas merecen atención. `Creating Schema History table` es Flyway creando su propia tabla de control la primera vez. Y `<< Empty Schema >>` es la comprobación de que el esquema estaba vacío: si no lo hubiera estado, esto habría fallado, y eso lleva directamente a `baseline` (más abajo). Ejecuta el mismo comando otra vez y responderá `Schema [dbo] is up to date. No migration necessary.` sin tocar nada: esa idempotencia es lo que permite meter `migrate` en un pipeline sin condicionales.

## La convención de nombres, desglosada

El nombre del fichero **no es decorativo**: es la única fuente de la que Flyway saca el tipo, el orden y la descripción de la migración. Anatomía completa:

```
V3__anadir_columna_stock_minimo.sql
│ │  │                          └── sufijo: .sql
│ │  └── descripción (los guiones bajos se muestran como espacios)
│ └── separador: DOS guiones bajos, obligatorio
└── prefijo (V) + versión (3)
```

La versión admite puntos como separadores de nivel (`V1`, `V2.1`, `V20260728.1`) y Flyway compara nivel a nivel como números, no como texto: `V10` va después de `V9`, al contrario de lo que pasaría en un orden alfabético.

Las tres familias de prefijo:

| Prefijo | Qué hace | Cuándo se aplica | Para qué sirve |
|---|---|---|---|
| `V` (versionada) | Se aplica **una sola vez**, en orden de versión | Si su versión es mayor que la actual del esquema | Cambios irreversibles: `CREATE TABLE`, `ALTER TABLE`, migraciones de datos |
| `R` (repetible) | Se **reaplica** cada vez que cambia su contenido | Después de todas las versionadas pendientes, en orden alfabético de descripción | Objetos que se recrean sin pérdida: vistas, procedimientos, funciones, *triggers* |
| `U` (undo) | Deshace la `V` de la misma versión | Solo con el comando `undo`, y de la más nueva a la más vieja | Rollback explícito de una migración concreta |

Dos aclaraciones que ahorran disgustos:

- **`U` (undo) es de la edición de pago.** En la Community Edition el comando `undo` no existe: responde `ERROR: Flyway Teams Edition upgrade required: undo is not supported by Flyway Community Edition`. Si tu plan de rollback depende de esto, cuéntalo como coste de licencia, no como una función que tienes.
- **Las repetibles no llevan número de versión**, y por eso su nombre tiene el doble guion bajo justo tras la `R`: `R__vista_productos_activos.sql`. Flyway guarda su checksum y en cada `migrate` compara: si el fichero cambió, lo reejecuta; si no, lo salta. Su contenido debe ser idempotente —`CREATE OR ALTER VIEW ProductosActivos AS ...`, nunca `CREATE VIEW` a secas— y solo debe describir objetos que se puedan recrear sin pérdida. Una tabla no lo es: "recrearla" significa tirar sus datos.

## `flyway_schema_history`: el historial

Toda la memoria de Flyway está en una tabla del propio esquema. Conviene mirarla, porque cuando algo va mal la respuesta está ahí.

```sql
SELECT installed_rank, version, description, checksum,
       installed_on, execution_time, success
FROM dbo.flyway_schema_history
ORDER BY installed_rank;
```

```
 installed_rank | version | description                 |  checksum   |    installed_on     | execution_time | success
----------------+---------+-----------------------------+-------------+---------------------+----------------+---------
              1 | 1       | crear tabla productos       |  1889301744 | 2026-07-28 09:14:02 |             41 |       1
              2 | 2       | crear tabla pedidos         | -1204887613 | 2026-07-28 09:14:02 |             58 |       1
              3 | 3       | anadir columna stock minimo |   842015377 | 2026-07-28 09:14:02 |             19 |       1
              4 |         | vista productos activos     |   -33091206 | 2026-07-28 09:14:02 |             12 |       1
```

`installed_rank` es el orden **real** de ejecución, que no tiene que coincidir con el de `version`: si algo se aplicó fuera de orden, aquí se ve. `version` es `NULL` en las repetibles, y eso es lo que las distingue en la tabla (fila 4). `checksum` es el hash del contenido del fichero al aplicarlo, base de toda la validación. Y `execution_time`, en milisegundos, dice qué migración va a ser la que duela cuando la tabla tenga diez millones de filas.

**Una fila con `success = 0` es una migración que empezó y falló.** Flyway la deja anotada a propósito, y a partir de ese momento **bloquea cualquier `migrate` posterior**: no sabe en qué estado quedó el esquema, así que se niega a seguir construyendo encima. Esa fila hay que quitarla con `repair` una vez arreglado el desaguisado.

## Los comandos del día a día

| Comando | Qué hace | Cuándo lo usas |
|---|---|---|
| `migrate` | Aplica las pendientes | Siempre; es el 95 % del uso |
| `info` | Lista estado de cada migración | Antes de cualquier despliegue dudoso |
| `validate` | Comprueba checksums y que no falte nada | En CI, como puerta antes del despliegue |
| `baseline` | Marca el estado actual como punto de partida | Al adoptar Flyway en una base ya existente |
| `repair` | Arregla el historial (no el esquema) | Tras una migración fallida o una corrección aprobada |
| `clean` | **Borra todo el esquema** | Solo en local, y con miedo |

`info` es el que más se infravalora. Muestra una tabla con el estado de cada fichero:

```
+-----------+---------+-----------------------------+------+---------------------+---------+
| Category  | Version | Description                 | Type | Installed On        | State   |
+-----------+---------+-----------------------------+------+---------------------+---------+
| Versioned | 1       | crear tabla productos       | SQL  | 2026-07-28 09:14:02 | Success |
| Versioned | 2       | crear tabla pedidos         | SQL  | 2026-07-28 09:14:02 | Success |
| Versioned | 3       | anadir columna stock minimo | SQL  | 2026-07-28 09:14:02 | Success |
| Repeatable|         | vista productos activos     | SQL  | 2026-07-28 09:14:02 | Success |
| Versioned | 4       | indice pedidos por cliente  | SQL  |                     | Pending |
+-----------+---------+-----------------------------+------+---------------------+---------+
```

`Pending` es lo que se aplicaría si ejecutaras `migrate` ahora. Otros estados que verás: `Failed` (la fila con `success = 0`), `Outdated` (una repetible cuyo fichero cambió), `Ignored` y `Out of Order`. Ejecutar `info` antes de desplegar cuesta dos segundos y te dice exactamente qué va a pasar.

## `baseline`: adoptar Flyway en una base de datos que ya existe

Esta es la primera pregunta de cualquiera que llega a un proyecto en marcha, y la que la documentación esconde. `TiendaDB` ya tiene `Productos`, `Pedidos` y `Clientes` creadas a mano hace dos años. Si lanzas `migrate`:

```
ERROR: Found non-empty schema(s) "dbo" but no schema history table. Use baseline() or set baselineOnMigrate to true to initialize the schema history table.
```

Flyway se niega, y hace bien: sus `V1`, `V2` y `V3` intentarían crear tablas que ya están. `baseline` resuelve esto declarando "el estado actual es la versión N; no intentes aplicar nada anterior":

```bash
docker run --rm -v "$(pwd)/sql:/flyway/sql" flyway/flyway \
  -url="jdbc:sqlserver://host.docker.internal:1433;databaseName=TiendaDB;encrypt=false" \
  -user=sa -password='Clave.Segura.2026' \
  -baselineVersion=3 -baselineDescription="esquema heredado" baseline
```

```
Creating Schema History table [TiendaDB].[dbo].[flyway_schema_history] with baseline ...
Successfully baselined schema with version: 3
```

Solo inserta **una fila** de tipo `BASELINE` con versión 3. A partir de ahí, `migrate` ignora todo lo que sea `V3` o inferior y aplica de `V4` en adelante. Lo habitual es volcar el esquema existente a un `V1__esquema_inicial.sql` —útil para crear entornos nuevos desde cero— y hacer `baseline -baselineVersion=1` en los entornos que ya estaban. Existe también `baselineOnMigrate=true`, que lanza el `baseline` automáticamente si encuentra el esquema no vacío: es cómodo y peligroso a la vez, porque en un entorno donde *sí* querías aplicar todo desde cero silencia el error y marca migraciones como aplicadas sin haberlas ejecutado.

## `repair`: cuándo arregla y cuándo tapa

`repair` **no toca el esquema**. Solo hace dos cosas en `flyway_schema_history`: borra las filas con `success = 0` y recalcula los checksums de las filas contra los ficheros actuales.

✅ Es la respuesta correcta cuando una migración falló, **ya has dejado la base de datos en un estado coherente** (a mano si hizo falta) y quieres desbloquear el historial para reintentar. Y cuando corregiste un fichero ya aplicado y el cambio es equivalente en efecto —un comentario, una tilde, el formato— con la corrección aprobada por el equipo.

❌ Es tapar el problema cuando el fichero cambió de verdad y el esquema de los entornos donde ya se aplicó **no** refleja el contenido nuevo. Ahí `repair` silencia la alarma y deja los entornos divergiendo: producción con una tabla y tu local con otra, y el historial jurando que están iguales. Igual de mal usarlo sobre una migración fallida sin haber comprobado antes qué llegó a ejecutarse.

Cuando termina, la segunda línea de su salida es el aviso importante, y es literal: `Manual cleanup of the remaining effects of the failed migration may still be required.` Flyway te está diciendo que el esquema es tu problema, no el suyo.

## `clean`: el comando que borra el esquema

`clean` elimina todos los objetos del esquema configurado —tablas, vistas, procedimientos, secuencias— y responde con un lacónico `Successfully cleaned schema [dbo] (execution time 00:00.187s)`. Deja `TiendaDB` como recién creada. Es genuinamente útil en local: reconstruir el esquema desde cero verifica que la cadena completa de migraciones funciona sobre una base vacía, que es lo que pasará en el próximo entorno nuevo.

En cualquier otro sitio es una catástrofe. Por eso las versiones recientes traen **`cleanDisabled=true` activado por defecto**, y el comando responde `ERROR: Unable to execute clean as it has been disabled with the 'flyway.cleanDisabled' property.` Deja siempre esa propiedad en `true` en la configuración de producción, de forma explícita, y no dependas del valor por defecto. Si no lo está, basta con que alguien confunda dos ficheros de configuración, o que un pipeline reciba un parámetro equivocado, para que `clean` borre `Productos`, `Pedidos` y `Clientes` sin preguntar. No hay confirmación interactiva y no hay `undo`: la recuperación es restaurar una copia de seguridad, con la pérdida de datos que haya entre el respaldo y ese momento.

## Configuración y precedencia

Los mismos ajustes se pueden dar de tres formas, y esto es lo que confunde cuando "el parámetro no se aplica":

```conf
# flyway.conf — lo estable y compartido, versionado en el repositorio
flyway.url=jdbc:sqlserver://localhost:1433;databaseName=TiendaDB;encrypt=false
flyway.locations=filesystem:sql
flyway.cleanDisabled=true
```

```bash
export FLYWAY_PASSWORD='Clave.Segura.2026'   # variable de entorno: las credenciales
flyway -target=4 migrate                     # línea de comandos: esta ejecución concreta
```

**Orden de precedencia, de menor a mayor: fichero `flyway.conf` → variables de entorno → parámetros de línea de comandos.** Lo último gana, y ese es el reparto práctico: lo compartido en el fichero, la contraseña siempre en el entorno —nunca en el `.conf`, que acaba en el repositorio— y lo puntual en la invocación.

### Placeholders: el mismo SQL en varios entornos

Un *placeholder* es un valor que Flyway sustituye dentro del SQL antes de ejecutarlo. Sirve para lo que cambia entre entornos y no se puede parametrizar en SQL: nombres de esquema, de usuario, de fichero.

```sql
-- V5__permisos_lectura_informes.sql
GRANT SELECT ON ${nombreEsquema}.Pedidos TO ${usuarioInformes};
```

Con `flyway.placeholders.nombreEsquema=dbo` y `flyway.placeholders.usuarioInformes=svc_informes_prod`, lo que llega de verdad a la base de datos es `GRANT SELECT ON dbo.Pedidos TO svc_informes_prod;`. Un detalle que sorprende: **el checksum se calcula sobre el fichero, no sobre el SQL resultante**, así que cambiar el valor de un placeholder no rompe la validación. Y si el SQL contiene un `${` literal que no quieres sustituir, Flyway falla con `No value provided for placeholder`; se resuelve con `flyway.placeholderReplacement=false` o cambiando los delimitadores.

## Los checksums y la validación

Al aplicar una migración, Flyway guarda el hash de su contenido. En cada `migrate` posterior compara el hash guardado con el del fichero actual. Lo que rompe la validación es **cualquier** diferencia byte a byte: un espacio al final de una línea, un comentario añadido, un salto de línea que pasó de `LF` a `CRLF` al clonar el repositorio en Windows.

```
ERROR: Validate failed: Migrations have failed validation. Migration checksum mismatch for migration version 3
-> Applied to database : 842015377
-> Resolved locally    : -1517420088
Either revert the changes to the migration, or run repair to update the schema history.
```

El mensaje ofrece dos salidas, y ninguna es siempre la correcta. El árbol de decisión real:

| ¿La migración ya se aplicó...? | Qué hacer | Por qué |
|---|---|---|
| ...en un entorno compartido (CI, *staging*, producción) | **Añade una migración nueva** (`V6__...`) | Ese esquema ya tiene el cambio antiguo; solo otra migración lo corrige de verdad |
| ...solo en tu local | `clean` y `migrate` de nuevo | El fichero corregido se aplica desde cero, sin historial que cuadrar |
| ...y la corrección es equivalente (formato, comentario, tilde) y está aprobada | `repair` | El esquema no cambia, solo hay que actualizar el hash |

La regla que resume las tres filas: **una migración `V` aplicada en un entorno compartido es inmutable**. Lo que rompe el trabajo del equipo no es el checksum, es haber editado el fichero.

## Migraciones que fallan a medias: el motor importa

Aquí Flyway hace lo mismo en todas partes, pero el resultado es muy distinto según el motor, y eso **cambia cómo escribes las migraciones**. Flyway envuelve cada una en una transacción: si el motor soporta **DDL transaccional** —PostgreSQL lo hace— una migración de cinco sentencias que falla en la cuarta se revierte entera, la base de datos queda como estaba y basta corregir el fichero y reintentar.

En **MySQL y MariaDB no hay DDL transaccional**: cada `CREATE`/`ALTER` confirma implícitamente. Una migración así:

```sql
-- V6__reorganizar_pedidos.sql
ALTER TABLE Pedidos ADD COLUMN CanalVenta VARCHAR(20) NOT NULL DEFAULT 'web';
ALTER TABLE Pedidos ADD COLUMN Descuento DECIMAL(10,2) NOT NULL DEFAULT 0;
CREATE INDEX idx_pedidos_canal ON Pedidos (CanalVena);   -- typo: falla aquí
```

deja `CanalVenta` y `Descuento` **creadas** y el índice sin crear, con una fila `success = 0` en el historial. Reintentar el fichero corregido falla de nuevo: `Duplicate column name 'CanalVenta'`. La secuencia correcta es inspeccionar qué llegó a aplicarse, deshacerlo a mano, `repair` para borrar la fila fallida y solo entonces `migrate`.

De ahí dos consecuencias prácticas cuando el motor no tiene DDL transaccional: **una sentencia por migración** cuando el cambio sea arriesgado (si `V6` hubieran sido tres ficheros, el fallo dejaría dos aplicadas limpiamente y una pendiente, sin nada que arreglar a mano), y **DDL tolerante** (`IF NOT EXISTS` donde el motor lo permita) para que reintentar sea inocuo. Nada de esto depende de la edición de Flyway: es una propiedad del motor, y conviene comprobarla antes de decidir cómo troceas las migraciones.

## Migraciones fuera de orden

El caso es real y pasa en cuanto hay dos ramas vivas. Ana crea `V13__anadir_canal_venta.sql` y Luis, en paralelo, `V14__indice_pedidos_cliente.sql`. La de Luis se fusiona y se despliega primero: el esquema queda en la versión 14. Cuando entra la de Ana, `migrate` responde `Current version of schema [dbo]: 14` y `Schema [dbo] is up to date. No migration necessary.`, e `info` la marca como `Ignored`.

Flyway **nunca aplica por defecto una versión inferior a la actual**, y la consecuencia es la peor posible: el despliegue termina en verde y la columna `CanalVenta` no existe. El código de Ana falla en producción con un error de columna inexistente.

Dos respuestas, según el entorno:

- **`-outOfOrder=true`** — aplica las pendientes rezagadas y las anota con un `installed_rank` posterior; la marca `Out of Order` queda en `info` para siempre. Es lo correcto en desarrollo y CI, donde las ramas se cruzan a diario.
- **Renumerar el fichero** a `V15__anadir_canal_venta.sql` antes de fusionar. Preferible cuando el conflicto se detecta a tiempo: el historial queda lineal y no depende de un flag.

Y si dos ramas eligen el mismo número, el fallo es inmediato y explícito:

```
ERROR: Found more than one migration with version 13
Offenders:
-> /flyway/sql/V13__anadir_canal_venta.sql (SQL)
-> /flyway/sql/V13__ampliar_direccion_cliente.sql (SQL)
```

Muchos equipos evitan la colisión usando *timestamps* como versión (`V20260728091402__...`), que en la práctica no chocan nunca.

## Callbacks

Un *callback* es un script que Flyway ejecuta en un momento concreto del ciclo, no como migración: basta poner `beforeMigrate.sql` o `afterMigrate.sql` en la carpeta de migraciones para que se ejecute en cada `migrate`, sin checksum ni historial. El caso más útil es `afterMigrate.sql` para recalcular estadísticas o reconstruir permisos tras cualquier cambio de esquema — algo que quieres que pase siempre, no una sola vez.

## Integración en un pipeline

El paso de migración va **antes** de desplegar la aplicación, como una etapa propia que puede fallar y detener el despliegue:

```yaml
# .github/workflows/despliegue.yml (fragmento)
- name: Aplicar migraciones
  run: |
    docker run --rm -v "${{ github.workspace }}/sql:/flyway/sql" flyway/flyway \
      -url="${{ secrets.TIENDA_DB_URL }}" -user="${{ secrets.TIENDA_DB_USER }}" \
      -password="${{ secrets.TIENDA_DB_PASSWORD }}" -cleanDisabled=true migrate

- name: Desplegar la API
  run: docker compose up -d --wait api
```

La alternativa tentadora —que cada instancia de la API ejecute `migrate` al arrancar— falla por tres razones concretas:

1. **Concurrencia.** Con tres instancias arrancando a la vez, las tres intentan migrar. Flyway usa un bloqueo en la base de datos, así que no se corrompe nada, pero dos se quedan esperando y el arranque se alarga o expira.
2. **Permisos.** Aplicar DDL exige un usuario que pueda modificar el esquema. Si la API lo lleva en su cadena de conexión, ese permiso está disponible en tiempo de ejecución para quien comprometa el proceso. Como paso de pipeline, solo existe durante la migración.
3. **Fallo tardío.** Si la migración falla durante el arranque, ya has empezado a sustituir instancias. Como etapa previa, el despliegue simplemente no ocurre y el sistema sigue en la versión anterior, intacto.

Para cambios que no pueden aplicarse con la aplicación en marcha, el paso previo no basta: eso se resuelve con las técnicas de [Estrategias zero-downtime](Estrategias-Zero-Downtime.md).

## Errores frecuentes

| Síntoma (mensaje literal) | Causa |
|---|---|
| `Found non-empty schema(s) "dbo" but no schema history table` | El esquema ya tenía objetos antes de usar Flyway; falta `baseline` |
| `Migration checksum mismatch for migration version 3` | Se editó un fichero ya aplicado — a veces solo un espacio o un `CRLF` por el clonado en Windows |
| `Found more than one migration with version 13` | Dos ramas eligieron el mismo número de versión |
| `Schema [dbo] is up to date. No migration necessary.` con una migración nueva sin aplicar | Su versión es inferior a la actual: llegó fuera de orden y se ignoró. Ver `info` (`Ignored`) y `outOfOrder` |
| `Detected failed migration to version 6` y ningún `migrate` avanza | Fila con `success = 0` en el historial; arregla el esquema y luego `repair` |
| Un fichero nuevo no aparece en `info` **y no hay error** | El nombre lleva **un** guion bajo (`V4_indice_pedidos.sql`) en lugar de dos; Flyway no lo reconoce como migración y lo ignora en silencio |
| `Incorrect string value: '\xC3\xB3n...'` o tildes convertidas en basura | El fichero no está en la codificación que Flyway espera (UTF-8 por defecto); fija `flyway.encoding` o guarda el `.sql` en UTF-8 |
| `Unable to execute clean as it has been disabled` | `cleanDisabled=true`. En producción esto es la protección funcionando, no un problema |

El del guion bajo simple es el más traicionero, precisamente porque no da error: el despliegue pasa en verde y el cambio no está. Si una migración "no se aplica" y `info` no la menciona, revisa el nombre antes que nada.

## Cuándo NO usar Flyway

- **El proyecto ya usa EF Core como ORM.** Añadir Flyway significa mantener dos herramientas y dos tablas de control para el mismo esquema. Con [EF Core Migrations](EF-Core-Migrations.md) las migraciones se generan a partir del modelo y viajan con el código; a menos que otro servicio en otro lenguaje comparta la base de datos, no hay razón para sumar Flyway.
- **Quieres que aplicar migraciones sea código C# de tu propio proceso.** Flyway es una herramienta externa que se invoca. Si prefieres una librería que se llama desde tu `Program.cs`, con tus logs y los scripts como recursos embebidos, eso es [DbUp](DbUp.md).
- **Necesitas rollback automático sin pagar.** Las migraciones `U` requieren edición comercial. En la Community Edition, deshacer es escribir una migración nueva que revierta, con los mismos cuidados que en cualquier estrategia forward-only.

## Buenas prácticas avanzadas

- **Trata una migración `V` aplicada fuera de tu local como código inmutable, y protégelo en CI.** El checksum detecta la edición, pero cuando lo hace ya hay entornos divergiendo. Un paso de `flyway validate` en cada *pull request* mueve el aviso al momento en que aún es gratis arreglarlo. Y añade a la revisión la pregunta explícita: ¿este fichero ya se ejecutó en algún sitio?
- **Reconstruye el esquema desde cero en CI, no solo apliques lo pendiente.** Un `clean` + `migrate` sobre una base vacía en cada build valida la cadena **completa** de migraciones. Es la única forma de descubrir que la `V7` depende de una columna que la `V12` renombró, un fallo que en los entornos existentes nunca se manifiesta y que reventará el día que crees un entorno nuevo.
- **Mide `execution_time` en un volumen realista antes de tocar producción.** Consulta `SELECT version, execution_time FROM flyway_schema_history ORDER BY execution_time DESC`: un `ALTER TABLE` que tarda 40 ms en tu `Pedidos` de mil filas puede bloquear la tabla varios minutos con diez millones. La migración que hay que trocear se identifica aquí, no en producción.
- **Ajusta el troceado de las migraciones al motor, no a tu gusto.** Con DDL transaccional (PostgreSQL) puedes agrupar varias sentencias relacionadas y confiar en el rollback. Sin él (MySQL), cada sentencia arriesgada merece su propio fichero, porque un fallo a medias deja un estado intermedio que hay que deshacer a mano antes de poder continuar.
- **Separa el usuario que migra del usuario que ejecuta la aplicación.** El de la API necesita `SELECT`/`INSERT`/`UPDATE`/`DELETE`; el de Flyway necesita crear y alterar objetos. Si son el mismo, cualquier inyección SQL en la aplicación puede hacer `DROP TABLE`. Es un cambio de cinco minutos que casi nadie hace.
- **Versiona los scripts en el mismo repositorio y el mismo *pull request* que el código que los necesita.** Así el cambio de esquema y su consumidor se revisan y despliegan juntos, y se evita el clásico "el código ya espera la columna nueva, pero la migración no se ha aplicado".

## Recursos didácticos

- [Documentación de Flyway (Redgate)](https://documentation.red-gate.com/flyway) — la referencia oficial. Empieza por *Concepts* y por la lista de parámetros de configuración; deja el resto para cuando lo necesites.
- [Imagen `flyway/flyway` en Docker Hub](https://hub.docker.com/r/flyway/flyway) — documenta los puntos de montaje (`/flyway/sql`, `/flyway/conf`) y qué drivers JDBC trae incluidos, que es lo primero que hace falta saber para el pipeline.
- [Flyway Community en GitHub](https://github.com/flyway/flyway) — el `CHANGELOG` es la forma más rápida de saber qué cambió de comportamiento entre versiones mayores, como `cleanDisabled` pasando a `true` por defecto.

---

*En resumen: Flyway aplica tus ficheros `.sql` una sola vez y en orden, y lo anota en `flyway_schema_history` — todo lo que tendrás que aprender después (`baseline`, `repair`, los checksums) sale de un único principio: una migración ya aplicada fuera de tu local no se toca.*
