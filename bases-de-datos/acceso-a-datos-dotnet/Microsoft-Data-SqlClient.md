# Microsoft.Data.SqlClient

## ¿Qué es?

Es el driver oficial de Microsoft para conectar aplicaciones .NET con SQL Server. Reemplaza al antiguo `System.Data.SqlClient` y es el paquete recomendado para cualquier proyecto .NET moderno que use SQL Server.

## ¿Por qué existe?

`System.Data.SqlClient` venía incluido en el framework de .NET pero quedó congelado: no recibe nuevas funcionalidades. `Microsoft.Data.SqlClient` nació como paquete NuGet independiente para poder iterar más rápido y soportar características modernas como Always Encrypted, autenticación con Azure Active Directory y conexiones resilientes. La diferencia práctica: si estás en .NET 6+, usas `Microsoft.Data.SqlClient`, punto.

## ¿Cuándo y para qué se usa?

En cualquier backend .NET que hable con SQL Server, este driver es la capa de transporte entre el código y la base de datos: una tienda online, un sistema de facturación o una app de gestión de tareas lo usan (directamente o a través de un ORM como Entity Framework Core) para ejecutar consultas y comandos contra sus tablas. Sin él, ninguna operación de persistencia funcionaría.

## Lo mínimo que necesitas saber

**1. Instalación**

```bash
dotnet add package Microsoft.Data.SqlClient --version 7.0.1
```

**2. Cadena de conexión en `appsettings.json`**

```json
"ConnectionStrings": {
  "DefaultConnection": "Server=localhost;Database=TiendaDB;Trusted_Connection=True;TrustServerCertificate=True;"
}
```

**3. Conexión y consulta básica**

```csharp
using Microsoft.Data.SqlClient;

await using var conn = new SqlConnection(connectionString);
await conn.OpenAsync();

await using var cmd = new SqlCommand("SELECT Id, Nombre FROM Productos WHERE Activo = 1", conn);
await using var reader = await cmd.ExecuteReaderAsync();

while (await reader.ReadAsync())
{
    Console.WriteLine($"{reader.GetInt32(0)} - {reader.GetString(1)}");
}
```

**4. Parámetros (evita SQL injection siempre)**

```csharp
cmd.CommandText = "SELECT * FROM Pedidos WHERE ProductoId = @id";
cmd.Parameters.AddWithValue("@id", productoId);
```

**5. Con Entity Framework Core**

Si usas EF Core, este driver trabaja por debajo de forma transparente. Lo configuras una sola vez en `Program.cs` y no vuelves a tocarlo directamente.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

## Lo que NO hace

- No es un ORM: no genera clases, no mapea objetos ni gestiona migraciones. Para eso existe Entity Framework Core.
- No maneja la lógica de reintentos automáticos por sí solo (aunque soporta resiliencia con políticas de Polly).
- No valida el esquema de la base de datos ni detecta cambios en tablas.

---

*En resumen: `Microsoft.Data.SqlClient` es el cable de red entre tu código C# y SQL Server — invisible cuando todo funciona, imprescindible siempre.*
