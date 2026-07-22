# Repositorios

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Un repositorio es una abstracción que hace que el dominio pueda cargar y guardar sus [agregados](Entidades-y-Agregados.md) como si trabajara con una colección en memoria, sin saber si por debajo hay una base de datos SQL, un fichero o una API externa.

## ¿Por qué existe?

El dominio necesita persistir y recuperar sus agregados, pero no debería saber nada de SQL, de un ORM ni de conexiones de red: eso rompería la [regla de dependencia](../clean-architecture/Regla-de-Dependencia.md). En el vocabulario de [Puertos y Adaptadores](../clean-architecture/Puertos-y-Adaptadores.md), un repositorio es exactamente un puerto de salida. Esta ficha no repite ese vocabulario general; se centra en las convenciones concretas del patrón repositorio cuando se aplica a agregados de dominio.

> Piensa en el repositorio como el mostrador de una biblioteca: le pides "el libro con este código" y te lo entrega, sin que tengas que saber en qué estantería física está ni cómo está catalogado por dentro.

## ¿Cuándo y para qué se usa?

Aparece siempre que un caso de uso necesita recuperar o guardar un agregado completo: un repositorio de pedidos que carga un `Pedido` con todas sus líneas, un repositorio de productos que guarda un `Producto` tras actualizar su stock. La regla clave: **un repositorio por raíz de agregado**, nunca uno por tabla ni uno por entidad interna.

## Lo mínimo que necesitas saber

**1. La interfaz vive en el dominio, con lenguaje de negocio**

```csharp
public interface IRepositorioPedidos
{
    Task<Pedido?> ObtenerPorId(int pedidoId);
    Task<IReadOnlyList<Pedido>> ObtenerPendientesDeEnvio();
    Task Guardar(Pedido pedido);
}
```

Los nombres de los métodos reflejan preguntas del negocio (`ObtenerPendientesDeEnvio`), no operaciones genéricas de base de datos.

**2. La implementación vive en la infraestructura**

```csharp
public class RepositorioPedidosSql(AppDbContext contexto) : IRepositorioPedidos
{
    public async Task<Pedido?> ObtenerPorId(int pedidoId) =>
        await contexto.Pedidos
            .Include(p => p.Lineas)
            .FirstOrDefaultAsync(p => p.Id == pedidoId);

    public async Task Guardar(Pedido pedido)
    {
        contexto.Update(pedido);
        await contexto.SaveChangesAsync();
    }
}
```

**3. Devuelve agregados completos, no filas sueltas ni DTOs**

`ObtenerPorId` entrega un `Pedido` con sus líneas ya cargadas, listo para que el caso de uso llame a sus métodos (`pedido.Confirmar()`). No devuelve un `DataRow` ni un objeto plano pensado para pintar en pantalla; para eso están las consultas de lectura, no el repositorio de escritura.

**4. Un repositorio por agregado, no por tabla**

Aunque `LineaPedido` tenga su propia tabla, no existe un `IRepositorioLineasPedido`: las líneas solo se alcanzan a través de `IRepositorioPedidos`, porque `LineaPedido` no es una raíz de agregado.

## Lo que NO hace

- **No es un envoltorio genérico de CRUD para cualquier tabla** — un repositorio pensado así deja de encapsular nada (ver buenas prácticas).
- **No sustituye a Dapper o Entity Framework Core** — el repositorio es la interfaz y la organización; por debajo sigue usando esas herramientas para hablar con la base de datos.
- **No expone `IQueryable` ni tipos del ORM en su contrato** — eso filtraría la tecnología de infraestructura hacia el dominio, la misma fuga que ya se señala en [Regla de Dependencia](../clean-architecture/Regla-de-Dependencia.md).

## Buenas prácticas avanzadas

- **Evita el repositorio genérico (`IRepository<T>`)** — parece ahorrar código, pero para servir a cualquier entidad termina exponiendo `IQueryable<T>` o un método `Find(Expression<...>)` que deja componer cualquier consulta desde fuera. Eso es justo lo que el patrón repositorio quería evitar: un repositorio específico por agregado, con métodos con nombre de negocio, protege mejor la frontera aunque tengas que escribir algo más de código.
- **El repositorio no es la unidad de trabajo** — `Guardar()` no debería hacer *commit* por su cuenta si el caso de uso coordina varios cambios; si cada `Guardar` confirma su propia transacción, un fallo a mitad de una operación deja datos a medias. Esto conecta con la buena práctica ya señalada en [Casos de Uso](../clean-architecture/Casos-de-Uso.md): el caso de uso es el dueño de la transacción, el repositorio solo participa en ella.
- **Separa lectura de escritura para los listados** — cargar el agregado completo (con validaciones e invariantes) solo para pintar una tabla en pantalla es coste innecesario. Una consulta de solo lectura, independiente del repositorio de escritura, que devuelva ya el DTO que necesita la pantalla, suele ser más simple y más rápida.
- **No dejes que el ORM cargue agregados ajenos "de propina"** — la carga perezosa (*lazy loading*) que cruza de un agregado a otro (por ejemplo, de `Pedido` a todo el histórico de `Cliente`) rompe el límite de consistencia y de transacción que definiste al diseñar los agregados. Carga explícitamente solo lo que el agregado necesita.

---

*En resumen: el repositorio deja que el dominio trate sus agregados como una colección en memoria — uno por raíz de agregado, con lenguaje de negocio, sin dejar que la tecnología de persistencia se asome por la interfaz.*
