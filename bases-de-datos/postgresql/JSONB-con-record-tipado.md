# JSONB con record tipado

## ¿Qué es?

Mapear una columna `jsonb` de PostgreSQL directamente a un `record` de C#, en vez de tratarla como una cadena de texto o un documento sin forma. El driver serializa y deserializa el contenido por ti; tú trabajas con un tipo con propiedades, no con claves de texto sueltas.

## ¿Por qué existe?

Una columna `jsonb` guarda datos con forma variable: el ejemplo típico es un producto con atributos que cambian según la categoría, talla en ropa, potencia en electrodomésticos. Pero en la práctica ese documento casi siempre tiene una forma conocida de antemano —la sabes tú, que escribes el código—, aunque la base de datos no la valide.

Sin mapeo, trabajar con ese documento en el backend significa ir a buscar cada campo por su clave de texto (`atributos["color"]`), sin autocompletado, sin comprobación en tiempo de compilación y con el riesgo de un error tipográfico que solo revienta en producción. Con un `record` tipado, el compilador avisa antes de desplegar.

> Si ya conoces los DTOs de una API REST, un `record` mapeado a `jsonb` es exactamente eso: un contrato tipado para un documento, solo que ese documento vive dentro de una fila en vez de viajar por HTTP.

## Habilitar el mapeo dinámico en Npgsql

Npgsql (el driver de PostgreSQL para .NET) necesita que le digas explícitamente qué tipos puede serializar como JSON. Se hace una vez, al construir el `NpgsqlDataSource`:

```csharp
var dataSourceBuilder = new NpgsqlDataSourceBuilder(connectionString);
dataSourceBuilder.EnableDynamicJson();
await using var dataSource = dataSourceBuilder.Build();
```

`EnableDynamicJson()` activa la conversión automática entre `jsonb`/`json` de PostgreSQL y cualquier tipo .NET que le pases como parámetro o que le pidas al leer, records incluidos. Sin esta llamada, Npgsql trata `jsonb` como texto plano y tendrías que serializar y deserializar a mano en cada consulta.

## El record que representa el documento

Define el documento como un `record` normal, con sus propiedades tipadas:

```csharp
public record AtributosProducto(string? Color, int? Talla);
```

Los dos campos son opcionales (`?`) porque, como el propio `jsonb` no valida nada, un documento antiguo puede no traerlos — más sobre esto en la siguiente sección.

Con Dapper, pasas y recuperas el record como si fuera cualquier otro parámetro:

```csharp
const string sql = "UPDATE productos SET atributos = @Atributos WHERE sku = @Sku";

await connection.ExecuteAsync(sql, new
{
    Sku = "ZAP-42",
    Atributos = new AtributosProducto(Color: "rojo", Talla: 42)
});
```

```csharp
const string select = "SELECT atributos FROM productos WHERE sku = @Sku";

var atributos = await connection.QuerySingleAsync<AtributosProducto>(select, new { Sku = "ZAP-42" });
// atributos.Color == "rojo", atributos.Talla == 42
```

Dapper no sabe nada de JSON: es Npgsql, por debajo, quien serializa el `record` al escribir y lo deserializa al leer. Dapper solo ve un parámetro de entrada y un valor de retorno.

## Qué pasa cuando el esquema del documento cambia

Aquí el enfoque se diferencia de una columna normal: no hay migración que aplicar sobre las filas existentes, porque `jsonb` no tiene columnas que alterar. Si mañana añades `Material` al catálogo, las filas antiguas simplemente no tienen esa clave en su documento.

```csharp
public record AtributosProducto(string? Color, int? Talla, string? Material = null);
```

Al deserializar un documento antiguo, `Material` llega a `null` sin que salte ningún error: `System.Text.Json` rellena con el valor por defecto lo que no encuentra. El riesgo va en la otra dirección: si **renombras** o **quitas** un campo del record, los documentos antiguos que aún usan el nombre viejo se leen con esa propiedad a `null`, silenciosamente, sin ningún aviso. No hay `ALTER TABLE` que te obligue a decidir qué hacer con los datos existentes.

## Errores frecuentes

| Síntoma | Causa habitual |
|---|---|
| `Can't write CLR type ... unless NpgsqlDataSourceBuilder.EnableDynamicJson() has been called` | Falta `EnableDynamicJson()` al construir el `NpgsqlDataSource`. |
| Una propiedad llega siempre a `null` aunque el JSON tiene el dato | El nombre de la propiedad no coincide con la clave del JSON: por defecto, `System.Text.Json` compara con distinción de mayúsculas salvo que fijes una `JsonNamingPolicy`. |
| El documento se guarda con las claves en otro orden del esperado | Normal: `jsonb` reordena las claves y normaliza el JSON al guardarlo, igual que hace en SQL. No afecta a la deserialización, solo a cómo se ve si inspeccionas la fila a mano. |

## Buenas prácticas avanzadas

- **Fija una `JsonNamingPolicy` explícita y única para todo el proyecto.** Si el resto de la aplicación serializa a `camelCase` (por ejemplo, para una API REST) pero el record de `jsonb` usa el `PascalCase` por defecto de C#, acabas con dos convenciones distintas para el mismo dato según por dónde lo mires. Configúralo una vez en las opciones de Npgsql y en las de la API, no lo dejes al azar.
- **Nunca renombres ni quites una propiedad sin plan de migración.** A diferencia de una columna SQL, quitar un campo de un record no falla: deserializa a `null` en silencio. Si un campo deja de usarse, márcalo obsoleto antes de borrarlo; si cambia de nombre, escribe un script que reescriba los documentos existentes con `UPDATE ... SET atributos = atributos || '{"nuevo": ...}'::jsonb`.
- **No metas en el documento lo que vayas a consultar, ordenar o relacionar con integridad.** Igual que en SQL, un campo dentro del record de `jsonb` no tiene clave foránea ni índice B-tree por defecto: si vas a filtrar por él con frecuencia, es una columna de la tabla, no una propiedad del documento.
- **En proyectos AOT o con arranque sensible al rendimiento, genera el serializador con `JsonSerializerContext`** (*source generation* de `System.Text.Json`) en vez de dejar que Npgsql use reflexión en cada mapeo: evita el coste de reflejar el tipo la primera vez que se usa.

## Documentación oficial

- [Npgsql: soporte de JSON](https://www.npgsql.org/doc/types/json.html) — cómo habilitar `EnableDynamicJson()`, qué tipos soporta y cómo combinarlo con `System.Text.Json`.
- [PostgreSQL: funciones y operadores JSON](https://www.postgresql.org/docs/current/functions-json.html) — la referencia completa de `->`, `->>`, `@>` y el resto de operadores que se pueden seguir usando desde SQL sobre el mismo documento.

---

*En resumen: un record tipado convierte un documento `jsonb` en un contrato con el que el compilador ayuda, a cambio de recordar que PostgreSQL sigue sin validar ni una coma de lo que hay dentro.*
