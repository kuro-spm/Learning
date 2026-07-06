# DbUp

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

**¿Qué es?** DbUp es una librería .NET ligera para aplicar migraciones SQL de forma programática: en vez de depender de una herramienta de línea de comandos externa, integras la aplicación de migraciones directamente en tu propio código C#, como un paso más de tu proceso de despliegue.

---

## ¿Por qué existe?

Herramientas como Flyway funcionan muy bien como CLI externa, pero algunos equipos .NET prefieren que aplicar las migraciones sea código C# normal: una consola de despliegue, una tarea de arranque o un paso de un pipeline que simplemente llama a una librería, sin instalar ni invocar una herramienta aparte. DbUp cubre ese hueco: es solo un paquete NuGet más, y tú decides desde dónde y cuándo se ejecuta.

---

## ¿Cuándo y para qué se usa?

Encaja bien en proyectos .NET que quieren migraciones basadas en SQL plano (como Flyway) pero prefieren evitar una dependencia externa a la línea de comandos: por ejemplo, una consola de despliegue interna que, antes de arrancar la API de un sistema de pedidos, aplica las migraciones pendientes contra la base de datos del entorno de destino.

---

## Lo mínimo que necesitas saber

**1. Los scripts SQL se organizan igual que en Flyway**

```
Scripts/
├── Script0001_CrearTablaProductos.sql
└── Script0002_AnadirColumnaStockMinimo.sql
```

**2. Se aplican con unas pocas líneas de C#**

```csharp
var resultado = DeployChanges.To
    .SqlDatabase(connectionString)
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly())
    .LogToConsole()
    .Build()
    .PerformUpgrade();

if (!resultado.Successful)
{
    Console.WriteLine(resultado.Error);
    return -1;
}
```

**3. Los scripts se pueden incrustar como recursos embebidos**

`WithScriptsEmbeddedInAssembly` lee los ficheros `.sql` marcados como *Embedded Resource* en el proyecto, así el ensamblado compilado ya lleva consigo todas las migraciones sin depender de rutas de fichero en el servidor de destino.

---

## Lo que NO hace

- **No genera SQL a partir de un modelo de clases** — igual que Flyway, cada script se escribe a mano.
- **No tiene interfaz de línea de comandos propia** — todo se orquesta desde tu propio código C#.

---

*En resumen: DbUp es Flyway con la aplicación de migraciones convertida en código C#, para equipos .NET que prefieren no salir de su propio ecosistema para desplegar cambios de esquema.*
