# PostgreSQL — Guía de tecnologías

Introducción a **PostgreSQL**, la base de datos relacional de código abierto, pensada para quien ya conoce SQL (tablas, filas, `JOIN`, transacciones) pero no ha trabajado con este motor en concreto. Explica qué la distingue de otras relacionales, cuándo conviene elegirla y qué capacidades trae de serie que en otros motores no encontrarás.

No repite los fundamentos de SQL: parte de lo que ya sabes y se centra en lo propio de Postgres (tipos ricos, `JSONB`, extensiones, índices GIN, MVCC), con ejemplos genéricos (una tienda online, usuarios, pedidos).

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [PostgreSQL](PostgreSQL.md) | Qué distingue a Postgres, tipos de datos, `JSONB`, claves foráneas, índices, transacciones, extensiones y buenas prácticas de operación. |
| 2 | [JSONB con record tipado](JSONB-con-record-tipado.md) | Cómo mapear una columna `jsonb` a un `record` de C# en el backend con Npgsql, en vez de tratarla como texto o un documento sin forma. |

---

> Para acceder a Postgres desde una app .NET, mira [Acceso a datos en .NET](../acceso-a-datos-dotnet/README.md); para versionar su esquema, [Migraciones de esquema](../migraciones-de-esquema/README.md); y para el enfoque documental (NoSQL), [MongoDB](../mongodb/README.md).
