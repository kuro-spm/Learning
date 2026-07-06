# Tenant Context

**¿Qué es?** Es el objeto que guarda "qué tenant es el de la petición actual" y lo deja disponible para todo el código que se ejecuta durante esa petición, sin tener que pasarlo a mano por cada método. Es el resultado de la [identificación del tenant](Identificacion-del-Tenant.md), almacenado para reutilizarlo.

---

## ¿Por qué existe?

Una vez que el middleware ha averiguado a qué tenant pertenece la petición, ese dato lo necesita *todo el mundo*: el repositorio que consulta la base de datos, el servicio que aplica reglas de negocio, el código que genera ficheros... Pasar el `TenantId` como parámetro en cada llamada sería repetitivo y fácil de olvidar.

El tenant context resuelve esto guardando el tenant actual en un sitio accesible durante toda la petición. Cada request tiene su propio contexto: dos peticiones simultáneas de tenants distintos no se pisan, porque el framework crea un *scope* aislado por petición.

> Si conoces el patrón de inyección de dependencias, piensa en el tenant context como un servicio *scoped* (uno por petición) que cualquier clase puede pedir en su constructor.

---

## ¿Cuándo y para qué se usa?

Lo usa cualquier capa que necesite saber "¿de quién son estos datos?": repositorios que filtran por `TenantId`, servicios que eligen la cadena de conexión, o lógica que aplica límites según el plan del tenant. Es la fuente única de verdad sobre el tenant durante la petición.

---

## Lo mínimo que necesitas saber

**1. Una interfaz sencilla que expone el tenant actual**

```csharp
public interface ITenantContext
{
    Tenant Current { get; }
}
```

**2. Se rellena una vez (desde el middleware) y se lee muchas**

```csharp
public class TenantContext : ITenantContext
{
    public Tenant Current { get; set; }   // lo fija el middleware al inicio
}
```

**3. Cualquier servicio lo recibe por inyección de dependencias**

```csharp
public class OrderRepository
{
    private readonly ITenantContext _tenant;

    public OrderRepository(ITenantContext tenant) => _tenant = tenant;

    public Task<List<Order>> GetAllAsync() =>
        _query.Where(o => o.TenantId == _tenant.Current.Id).ToListAsync();
}
```

**4. Es *scoped*, no global**

Se registra con vida "por petición" (`AddScoped`), no como singleton. Así cada request tiene su propio tenant y no hay riesgo de que una petición vea el tenant de otra.

```csharp
services.AddScoped<ITenantContext, TenantContext>();
```

---

## Lo que NO hace

- **No identifica el tenant** — solo lo *guarda*; quién lo calcula es la [resolución del tenant](Identificacion-del-Tenant.md).
- **No debe ser un singleton ni una variable estática global** — eso compartiría el tenant entre peticiones distintas y provocaría fugas de datos.
- **No filtra los datos por sí mismo** — es solo la fuente del `TenantId`; aplicarlo en las consultas es otro trabajo (ver [Aislamiento por Fila](Aislamiento-por-Fila.md)).

---

## Buenas prácticas avanzadas

- **No inyectes `ITenantContext` en un singleton** — es la *captive dependency* clásica: el singleton captura el contexto de la primera petición que lo crea y servirá **ese** tenant para siempre, a todos los clientes. El contenedor no siempre avisa. En un singleton, inyecta `IServiceScopeFactory` y resuelve el contexto dentro de un scope en cada uso.
- **Que `Current` falle a gritos, no devuelva `null`** — haz que el getter lance una excepción si el middleware aún no lo ha fijado. Un `null` silencioso se propaga hasta convertirse en una consulta que filtra por `Guid.Empty` (cero resultados, bug confuso) o, peor, que no filtra; una excepción en el primer acceso señala el punto exacto donde falta el contexto.
- **Separa quién lo escribe de quién lo lee** — si `Current` tiene un `set` público en la interfaz, cualquier servicio puede cambiar el tenant a mitad de petición. Expón `ITenantContext` de solo lectura al resto del código y deja la escritura en una interfaz aparte (`ITenantContextSetter`) que solo conozca el middleware.

---

*En resumen: el tenant context es la "memoria" de quién es el tenant durante una petición, accesible para todas las capas sin pasarlo a mano —y siempre por petición, nunca global.*
