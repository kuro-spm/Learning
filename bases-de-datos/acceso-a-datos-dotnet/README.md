# Acceso a datos en .NET — Guía de tecnologías

Cómo habla una aplicación .NET con una base de datos relacional, capa por capa: desde el driver que abre el socket contra SQL Server hasta el micro-ORM que convierte las filas de un `SELECT` en objetos C#.

Las dos fichas están pensadas para leerse en orden. La primera explica la infraestructura que siempre está debajo (aunque uses Entity Framework Core y no la veas); la segunda, la herramienta con la que escribirás las consultas a mano cuando quieras controlar el SQL.

El escenario que comparten los ejemplos es una tienda online con la base de datos `TiendaDB` y las tablas `Productos`, `Pedidos` y `Clientes`.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) | El driver oficial para SQL Server: cadena de conexión parámetro a parámetro, cifrado obligatorio desde la versión 4.0, connection pooling, parámetros tipados y errores transitorios. Es la base sobre la que corren EF Core y Dapper. |
| 2 | [Dapper](Dapper.md) | El micro-ORM que mapea resultados SQL a objetos C# sin generar el SQL por ti: qué método usar según las filas que esperes, multi-mapping con `splitOn`, transacciones y cuándo compensa frente a EF Core. |

---

## Qué resuelve cada capa

Una duda recurrente al empezar es cuál de las tres opciones toca en cada caso. Las tres pueden convivir en el mismo proyecto:

| Necesidad | Herramienta | Por qué |
|---|---|---|
| Abrir la conexión, cifrar el transporte, reintentar un error transitorio | `Microsoft.Data.SqlClient` | Es el driver: nadie más hace este trabajo. |
| Una consulta de lectura con joins y filtros que quieres controlar tú | Dapper | Escribes el SQL exacto y él solo mapea el resultado. |
| Escrituras sobre un modelo de dominio, con change tracking y migraciones | Entity Framework Core | Genera el SQL y mantiene el esquema. |
| Cargar decenas de miles de filas de golpe | `SqlBulkCopy` (en el driver) | Órdenes de magnitud más rápido que N `INSERT`. |

## Los errores que se pagan caros

| Síntoma | Causa habitual | Dónde se explica |
|---|---|---|
| `The certificate chain was issued by an authority that is not trusted` | `Encrypt=True` es el valor por defecto desde la versión 4.0 del driver | [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) |
| `Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool` | Conexiones que no se liberan, o una cadena de conexión distinta por petición | [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) |
| Una consulta indexada hace *scan* en lugar de *seek* | `AddWithValue` infiere el tipo y fuerza una conversión implícita | [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) |
| Propiedades que llegan a `null` sin que salte ningún error | El nombre de la columna no coincide con el de la propiedad | [Dapper](Dapper.md) |
| `ensure you set the splitOn param if you have keys other than Id` | Multi-mapping sin indicar dónde empieza la segunda entidad | [Dapper](Dapper.md) |

---

> Si buscas un ORM completo (que genera el SQL y gestiona las migraciones), echa un vistazo a [Entity Framework Core](../../desarrollo-web/de-wpf-a-web/Entity-Framework-Core.md). Y si el problema es que la misma consulta se repite cientos de veces por minuto, la solución probablemente no sea otro ORM sino [caching](../caching/README.md).
