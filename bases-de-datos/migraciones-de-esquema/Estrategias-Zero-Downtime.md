# Estrategias de Migración Zero-Downtime

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Son un conjunto de técnicas para cambiar el esquema de una base de datos en producción sin detener el servicio ni romper la compatibilidad con las instancias de la aplicación que siguen ejecutando la versión anterior del código durante el despliegue.

## ¿Por qué existe?

En un despliegue con varias instancias, durante una ventana de tiempo conviven la versión antigua y la nueva del código (mientras el balanceador va rotando instancias). Si una migración cambia el esquema de forma incompatible con la versión antigua —por ejemplo, borra una columna que el código viejo todavía usa—, esas instancias antiguas empiezan a fallar antes de que termine el despliegue. Las estrategias zero-downtime existen para que el esquema pueda cambiar sin que exista jamás un instante en el que el esquema y alguna versión del código en producción sean incompatibles entre sí.

> Es parecido a cambiar de carril en una autopista con el coche en marcha: no puedes saltar directamente del carril viejo al nuevo, tienes que solaparlos un rato mientras te desplazas de uno a otro sin salirte de la vía en ningún momento.

## ¿Cuándo y para qué se usa?

Cualquier sistema que se despliega sin ventana de mantenimiento (varias instancias, alta disponibilidad) y necesita cambios de esquema "delicados": renombrar una columna, cambiar su tipo, eliminarla, o dividir una tabla en dos. Ejemplos típicos: renombrar la columna `Nombre` de `Producto` a `NombreProducto` en una tienda online sin que el despliegue tenga un instante de error 500, o cambiar cómo se calcula el estado de un pedido sin que los pedidos en curso durante el despliegue se vean afectados.

## Lo mínimo que necesitas saber

**1. El patrón *expand/contract* (o *parallel change*)**

Divide un cambio "peligroso" en varios pasos pequeños, cada uno compatible con la versión de código anterior y la siguiente:

1. **Expand**: añade lo nuevo sin tocar lo viejo (por ejemplo, la columna nueva, además de la vieja).
2. **Migrate**: despliega el código que sabe usar ambas, y rellena/sincroniza los datos.
3. **Contract**: una vez todas las instancias usan solo lo nuevo, elimina lo viejo.

**2. Renombrar una columna, paso a paso**

```sql
-- Paso 1 (expand): añadir la columna nueva, sin tocar la vieja
ALTER TABLE Productos ADD NombreProducto NVARCHAR(200) NULL;

-- Copiar los datos existentes
UPDATE Productos SET NombreProducto = Nombre;
```

```csharp
// Paso 2 (migrate): el código nuevo escribe en ambas columnas durante la transición
producto.Nombre = nombre;
producto.NombreProducto = nombre;
```

```sql
-- Paso 3 (contract): una vez todo el código usa solo la columna nueva
ALTER TABLE Productos DROP COLUMN Nombre;
```

**3. Añadir una columna obligatoria sin romper inserts antiguos**

Una columna `NOT NULL` sin valor por defecto rompe cualquier `INSERT` de código que no la conozca todavía. La secuencia segura:

```sql
-- Paso 1: nullable primero
ALTER TABLE Pedidos ADD NumeroSeguimiento NVARCHAR(50) NULL;

-- Paso 2 (tras desplegar el código que siempre la rellena): forzar NOT NULL
ALTER TABLE Pedidos ALTER COLUMN NumeroSeguimiento NVARCHAR(50) NOT NULL;
```

**4. Eliminar una columna solo cuando ya nadie la lee**

El borrado de una columna es siempre el último paso, nunca el primero: primero se despliega el código que deja de usarla, se confirma que ninguna instancia antigua sigue en producción, y solo entonces se borra en una migración aparte.

**5. Índices y tablas grandes: crear sin bloquear**

En tablas con muchas filas, crear un índice o cambiar un tipo de columna puede bloquear la tabla durante minutos. La mayoría de motores ofrecen una variante *online* para evitarlo:

```sql
CREATE INDEX IX_Pedidos_ClienteId ON Pedidos(ClienteId) WITH (ONLINE = ON);
```

## Lo que NO hace

- **No elimina la necesidad de planificar el orden de despliegue** — sigue haciendo falta decidir explícitamente qué se despliega primero, el esquema o el código, y verificar que el paso intermedio es seguro.
- **No es gratis en complejidad** — un cambio que en una ventana de mantenimiento sería un solo script, aquí son varios pasos y varios despliegues; solo compensa cuando de verdad no puedes permitirte *downtime*.
- **No sustituye a las pruebas de compatibilidad hacia atrás** — cada paso intermedio debería probarse con la combinación real de esquema y código que va a convivir en producción durante la transición.

## Buenas prácticas avanzadas

- **Aplica siempre "nullable o con default" antes que "NOT NULL"** — añadir una columna obligatoria de golpe es la forma más común de romper un despliegue sin darse cuenta, porque falla justo en el instante en que conviven versiones de código.
- **Nunca borres y renombres en el mismo paso** — un `sp_rename` o un `RenameColumn` puede parecer atómico, pero si el código viejo sigue esperando el nombre anterior, revienta igual que si hubieras borrado la columna. Trata cualquier renombrado como un *expand/contract* completo.
- **Feature flags para el código, migraciones en fases para el esquema** — combinar ambos permite activar el uso del esquema nuevo en el código de forma independiente del propio despliegue, dando margen para revertir solo el flag si algo va mal, sin tocar la base de datos de nuevo.
- **Mide el tiempo de bloqueo de cada cambio en un entorno con volumen realista antes de aplicarlo en producción** — un `ALTER TABLE` que tarda 200 ms en tu base de datos local puede tardar minutos sobre una tabla de producción con millones de filas, y ese tiempo puede ser de bloqueo exclusivo.
- **Ten un paso de "contract" pendiente, no lo olvides** — es habitual completar el "expand" y el "migrate", dejar el sistema funcionando, y no volver nunca a limpiar las columnas u objetos viejos. Trátalo como una tarea de la misma migración, no como algo opcional para "cuando haya tiempo".

---

*En resumen: zero-downtime no es una herramienta sino una disciplina — dividir cada cambio de esquema en pasos que nunca dejan de ser compatibles con el código que convive con ellos durante el despliegue.*
