# Bases de datos — Guías

PostgreSQL como base de datos relacional, cómo acceder a una desde una aplicación .NET, mantenerla en caché y versionar su esquema a lo largo del tiempo, además de una introducción a las bases de datos documentales (NoSQL) con MongoDB.

---

## Contenido

### [PostgreSQL](postgresql/README.md)
La base de datos relacional de código abierto: qué la distingue, tipos ricos, `JSONB`, extensiones e índices.

### [Acceso a datos en .NET](acceso-a-datos-dotnet/README.md)
El driver de SQL Server y Dapper como micro-ORM.

### [Caching](caching/README.md)
Por qué y cómo cachear datos costosos de obtener: patrones, Redis y estrategias de invalidación.

### [Migraciones de esquema](migraciones-de-esquema/README.md)
Cómo versionar cambios en el esquema de la base de datos: EF Core Migrations, Flyway, DbUp y despliegues sin downtime.

### [MongoDB](mongodb/README.md)
La base de datos NoSQL orientada a documentos, explicada desde lo que ya sabes de SQL: colecciones, documentos, modelado y aggregation pipeline.
