# Acceso a datos en .NET — Guía de tecnologías

Cómo habla una aplicación .NET con una base de datos relacional, capa por capa: desde el driver que abre el socket contra SQL Server hasta el ORM que escribe el SQL por ti.

Las tres fichas están pensadas para leerse en orden. La primera explica la infraestructura que siempre está debajo, aunque no la veas; la segunda, la herramienta con la que escribirás las consultas a mano cuando quieras controlar el SQL; la tercera, el ORM completo que te lo genera y mantiene el estado de las entidades por ti.

El escenario que comparten los ejemplos es una tienda online con la base de datos `TiendaDB` y las tablas `Productos`, `Pedidos`, `LineasPedido` y `Clientes`, siguiendo el pedido **#4711**.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) | El driver oficial para SQL Server: cadena de conexión parámetro a parámetro, cifrado obligatorio desde la versión 4.0, connection pooling, parámetros tipados y errores transitorios. Es la base sobre la que corren las otras dos. |
| 2 | [Dapper](Dapper.md) | El micro-ORM que mapea resultados SQL a objetos C# sin generar el SQL por ti: qué método usar según las filas que esperes, multi-mapping con `splitOn`, transacciones y cuándo compensa frente a EF Core. |
| 3 | [Entity Framework Core](Entity-Framework-Core.md) | El ORM completo: el modelo y sus convenciones, el SQL que genera cada consulta LINQ, *change tracking*, las cuatro estrategias de carga de relaciones, el problema N+1 y cómo diagnosticarlo. |

---

## Qué resuelve cada capa

Una duda recurrente al empezar es cuál de las tres opciones toca en cada caso. Las tres pueden convivir en el mismo proyecto:

| Necesidad | Herramienta | Por qué |
|---|---|---|
| Abrir la conexión, cifrar el transporte, reintentar un error transitorio | `Microsoft.Data.SqlClient` | Es el driver: nadie más hace este trabajo. |
| Una consulta de lectura con joins y filtros que quieres controlar tú | Dapper | Escribes el SQL exacto y él solo mapea el resultado. |
| Escrituras sobre un modelo de dominio, con change tracking y migraciones | [Entity Framework Core](Entity-Framework-Core.md) | Genera el SQL y mantiene el esquema. |
| Cargar decenas de miles de filas de golpe | `SqlBulkCopy` (en el driver) | Órdenes de magnitud más rápido que N `INSERT`. |

## Los errores que se pagan caros

| Síntoma | Causa habitual | Dónde se explica |
|---|---|---|
| `The certificate chain was issued by an authority that is not trusted` | `Encrypt=True` es el valor por defecto desde la versión 4.0 del driver | [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) |
| `Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool` | Conexiones que no se liberan, o una cadena de conexión distinta por petición | [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) |
| Una consulta indexada hace *scan* en lugar de *seek* | `AddWithValue` infiere el tipo y fuerza una conversión implícita | [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) |
| Propiedades que llegan a `null` sin que salte ningún error | El nombre de la columna no coincide con el de la propiedad | [Dapper](Dapper.md) |
| `ensure you set the splitOn param if you have keys other than Id` | Multi-mapping sin indicar dónde empieza la segunda entidad | [Dapper](Dapper.md) |
| Una sola petición dispara cientos de consultas | N+1: se recorre una colección accediendo a una propiedad de navegación sin `Include` | [Entity Framework Core](Entity-Framework-Core.md) |
| `The LINQ expression could not be translated` | La consulta usa algo que el proveedor no sabe convertir en SQL | [Entity Framework Core](Entity-Framework-Core.md) |
| Una consulta devuelve las filas multiplicadas | Dos `Include` de colecciones en la misma consulta: producto cartesiano | [Entity Framework Core](Entity-Framework-Core.md) |

---

> Las tres pueden convivir en el mismo proyecto, y de hecho es lo habitual: EF Core para el modelo de dominio y las escrituras, Dapper para las consultas que LINQ no expresa bien, y el driver directamente para las cargas masivas. Lo que versiona el esquema es una capa aparte: [migraciones de esquema](../migraciones-de-esquema/README.md). Y si el problema es que la misma consulta se repite cientos de veces por minuto, la solución no es otro ORM sino [caching](../caching/README.md).
