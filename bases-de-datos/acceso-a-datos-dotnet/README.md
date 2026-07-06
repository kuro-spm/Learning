# Acceso a datos en .NET — Guía de tecnologías

Cómo habla una aplicación .NET con una base de datos relacional: desde el driver de bajo nivel hasta un micro-ORM que te ahorra escribir mapeos a mano.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) | El driver de bajo nivel que conecta .NET con SQL Server. Leer antes que Dapper. |
| 2 | [Dapper](Dapper.md) | El micro-ORM que mapea resultados SQL a objetos C# sin generar el SQL por ti. |

---

> Si buscas un ORM completo (que genera el SQL y gestiona migraciones), echa un vistazo a [Entity Framework Core](../../desarrollo-web/de-wpf-a-web/Entity-Framework-Core.md) en la guía de desarrollo web.
