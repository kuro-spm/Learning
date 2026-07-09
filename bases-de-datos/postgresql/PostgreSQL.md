# PostgreSQL

## ¿Qué es?

PostgreSQL (o "Postgres") es una base de datos **relacional** de código abierto: guarda los datos en tablas con filas y columnas y se consulta con SQL, como SQL Server o MySQL. Su seña de identidad es lo lejos que lleva ese modelo: cumple el estándar SQL con rigor y añade tipos de datos, extensiones y capacidades que la acercan a un motor "todoterreno".

## ¿Por qué existe?

Todas las bases de datos relacionales hacen lo mismo de base (tablas, `JOIN`, transacciones), así que la pregunta real no es "¿por qué una relacional?" sino "¿por qué *esta*?".

PostgreSQL nació en la universidad de Berkeley con la idea de ser una base de datos **extensible**: en vez de un conjunto cerrado de funciones, un núcleo al que se le pueden añadir tipos de datos, operadores e incluso lenguajes. Décadas después eso se traduce en un motor que respeta el estándar SQL casi al pie de la letra, es gratis y sin licencias por core, y trae de serie cosas que en otros motores son de pago o no existen: campos JSON indexables, búsqueda de texto completo, tipos geográficos, arrays como columna...

> Si vienes de SQL Server o MySQL, piensa en PostgreSQL como "una relacional muy parecida, pero más estricta con el estándar y con una caja de herramientas mucho más grande incluida".

## ¿Cuándo y para qué se usa?

Es una elección por defecto sensata para el almacén principal de casi cualquier aplicación: el catálogo y los pedidos de una tienda online, los usuarios y publicaciones de un blog, las tareas de una app de productividad. Brilla especialmente cuando los datos son relacionales pero **algún campo es semiestructurado** (guardar los atributos variables de un producto en una columna `JSONB` sin renunciar a las tablas para el resto), cuando necesitas **búsqueda de texto** sin montar un motor aparte, o cuando el proyecto usa **datos geográficos** (con la extensión PostGIS).

Es peor elección si tu caso es puramente de caché en memoria (ahí encaja Redis) o si los datos son tan flexibles y sin relaciones que una base documental como MongoDB te ahorraría trabajo de modelado.

## Lo mínimo que necesitas saber

**1. Conectarse y hablar SQL estándar**

El SQL del día a día es el que ya conoces. La herramienta de línea de comandos es `psql`:

```sql
CREATE TABLE products (
    id          SERIAL PRIMARY KEY,   -- entero autoincremental
    name        TEXT NOT NULL,
    price       NUMERIC(10,2),
    created_at  TIMESTAMPTZ DEFAULT now()
);

SELECT id, name FROM products WHERE price < 100 ORDER BY name;
```

**2. Tipos de datos ricos**

Postgres va mucho más allá de `VARCHAR` e `INT`: `TEXT` (cadena sin límite), `BOOLEAN`, `UUID`, `TIMESTAMPTZ` (fecha-hora con zona horaria), arrays (`TEXT[]`) y `JSONB` como columnas de primera clase.

```sql
CREATE TABLE users (
    id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tags   TEXT[],                    -- un array en una sola columna
    prefs  JSONB                      -- un objeto JSON indexable
);
```

**3. JSONB: lo relacional y lo flexible a la vez**

Puedes guardar un objeto JSON en una columna y consultarlo con operadores propios (`->`, `->>`, `@>`), sin renunciar a las tablas normales para el resto de datos:

```sql
SELECT name FROM products
WHERE attributes ->> 'color' = 'rojo';   -- ->> extrae el valor como texto
```

**4. Claves foráneas e integridad referencial**

A diferencia de una base documental, Postgres garantiza que una referencia apunta a algo que existe:

```sql
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    user_id     UUID REFERENCES users(id),   -- no se puede insertar un user_id inexistente
    total       NUMERIC(10,2)
);
```

**5. Índices para acelerar búsquedas**

Sin índice, filtrar recorre toda la tabla. Postgres ofrece varios tipos (B-tree por defecto; GIN para `JSONB` y arrays):

```sql
CREATE INDEX idx_products_name ON products (name);
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
```

**6. Transacciones**

Un bloque de operaciones que se confirma entero (`COMMIT`) o se deshace entero (`ROLLBACK`), nunca a medias:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**7. Extensiones**

Funcionalidad que se activa con una línea. Las más habituales: `pgcrypto` (funciones criptográficas), `uuid-ossp` (UUIDs) y **PostGIS** (datos geográficos).

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

## Lo que NO hace

- **No es una base de datos en memoria** — persiste en disco; para caché volátil de baja latencia, esa función la cumple mejor una herramienta como Redis.
- **No escala en horizontal por sí sola** — repartir la carga entre varios nodos (sharding) no viene de serie; requiere extensiones o arquitectura extra.
- **No valida tus JSONB por ti** — dentro de una columna `JSONB` puedes meter cualquier forma; la coherencia de ese contenido la garantizas tú, igual que en una base documental.
- **No es idéntica a otros motores** — el SQL estándar es portable, pero tipos como `SERIAL`/`JSONB` y las extensiones son propios; migrar desde/hacia SQL Server o MySQL tiene matices.

## Buenas prácticas avanzadas

- **Usa `TIMESTAMPTZ`, nunca `TIMESTAMP` a secas** — `TIMESTAMPTZ` guarda el instante normalizado a UTC y lo devuelve en la zona del cliente; `TIMESTAMP` guarda un número sin zona que "parece" una hora pero es ambiguo. Casi todos los bugs de fechas nacen de haber elegido el tipo sin zona por descuido.
- **Entiende el `VACUUM` y el modelo MVCC** — Postgres no borra las filas al instante: un `UPDATE` o `DELETE` deja "versiones muertas" que el proceso `autovacuum` limpia después. En tablas con mucha escritura, si el vacuum no da abasto la tabla se hincha (*bloat*) y se ralentiza; vigilarlo distingue a quien opera Postgres en serio.
- **Lee los planes con `EXPLAIN (ANALYZE, BUFFERS)`** — antes de añadir índices a ciegas, mira cómo ejecuta Postgres la consulta. `Seq Scan` sobre una tabla grande donde filtras es la señal clásica de que falta un índice; un índice que nunca aparece en los planes solo ralentiza las escrituras.
- **Elige el índice según el dato: GIN para `JSONB`/arrays/texto, B-tree para el resto** — indexar una columna `JSONB` con el B-tree por defecto no sirve para las consultas `@>` o `->>`; necesita un índice GIN. Usar el tipo equivocado es tener un índice que el planificador ignora.
- **No abuses del `JSONB` como excusa para no modelar** — es tentador meter todo en una columna JSON "por flexibilidad", pero los datos que consultas y relacionas de verdad merecen sus propias columnas y claves foráneas. Reserva `JSONB` para lo genuinamente variable o semiestructurado.
- **Prefiere `NUMERIC` a `FLOAT` para dinero** — `FLOAT`/`REAL` son binarios y arrastran errores de redondeo (`0.1 + 0.2 ≠ 0.3`); `NUMERIC(10,2)` es exacto. Para importes y cantidades contables, siempre `NUMERIC`.

## Recursos didácticos

La **documentación oficial** (<https://www.postgresql.org/docs/>) es una de las mejores que existen y se puede leer casi como un libro. Para practicar SQL contra una base Postgres real desde el navegador, sin instalar nada, **pg_sql** en <https://pgexercises.com/> propone ejercicios progresivos con solución. Y si quieres visualizar cómo el planificador ejecuta una consulta, pega la salida de `EXPLAIN` en <https://explain.dalibo.com/> y te la dibuja como un árbol navegable.

---

*En resumen: PostgreSQL es una base de datos relacional de código abierto muy fiel al estándar SQL y con una enorme caja de herramientas incluida (JSONB, arrays, extensiones) — la opción por defecto cuando quieres lo relacional de siempre sin quedarte corto de funciones.*
