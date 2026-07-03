# Aislamiento por Fila

## ¿Qué es?

Es la técnica para garantizar que, en una base de datos **compartida** entre tenants, cada consulta solo vea las filas de su propio tenant. Se consigue marcando cada fila con una columna `TenantId` y filtrando *siempre* por ella, idealmente de forma automática para no depender de la memoria del programador.

## ¿Por qué existe?

En la estrategia de base compartida (ver [Estrategias de Aislamiento](Estrategias-de-Aislamiento.md)), todos los tenants conviven en las mismas tablas. El aislamiento es solo "lógico": depende de que **toda** consulta incluya `WHERE TenantId = @actual`. El problema es humano: tarde o temprano alguien escribe una consulta y olvida el filtro. Esa única consulta filtrada de menos puede devolver datos de otros clientes.

> Es como una oficina compartida donde cada documento lleva el nombre de su empresa. Si te fías de que cada empleado lea siempre la etiqueta antes de coger un papel, un despiste basta para filtrar. Mejor poner un guardia automático que tape los documentos que no son tuyos.

El aislamiento por fila existe para que ese filtro no dependa de recordarlo: se aplica solo.

## ¿Cuándo y para qué se usa?

Siempre que uses una base de datos compartida con discriminador. Es la pieza de seguridad central de esa estrategia. Se implementa en dos niveles que se complementan: en la aplicación (filtros automáticos del ORM) y en la propia base de datos (Row-Level Security).

## Lo mínimo que necesitas saber

**1. La columna discriminadora en cada tabla**

```csharp
public class Order
{
    public Guid Id { get; set; }
    public Guid TenantId { get; set; }   // ← todas las tablas de negocio la llevan
    public decimal Total { get; set; }
}
```

**2. Filtro global automático en el ORM (EF Core *global query filters*)**

En lugar de añadir el `WHERE` a mano en cada consulta, le dices al ORM que lo añada **siempre**, de forma transparente.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // A partir de aquí, TODA consulta sobre Order filtra por el tenant actual
    modelBuilder.Entity<Order>()
        .HasQueryFilter(o => o.TenantId == _tenantContext.Current.Id);
}
```

**3. Asignar el TenantId al guardar, también automático**

Igual que se filtra al leer, conviene rellenar el `TenantId` al insertar, para que nadie tenga que acordarse.

```csharp
public override int SaveChanges()
{
    foreach (var entry in ChangeTracker.Entries<Order>())
        if (entry.State == EntityState.Added)
            entry.Entity.TenantId = _tenantContext.Current.Id;

    return base.SaveChanges();
}
```

**4. Row-Level Security (RLS): la red de seguridad en la base de datos**

RLS es una función del motor de base de datos (SQL Server, PostgreSQL...) que filtra las filas **dentro de la propia base**, según una variable de sesión. Aunque una consulta llegue sin filtro, la base solo devuelve las filas del tenant activo. Es la defensa final si la aplicación falla.

```sql
-- Solo se ven las filas cuyo TenantId coincide con la variable de sesión
CREATE SECURITY POLICY TenantFilter
ADD FILTER PREDICATE dbo.fn_tenant_predicate(TenantId) ON dbo.Orders
WITH (STATE = ON);
```

**5. Defensa en profundidad**

No te fíes de una sola capa. El filtro del ORM evita el 99 % de los errores; RLS cubre el 1 % restante (consultas crudas, herramientas externas, bugs). Las dos juntas son lo que hace seguro el modelo compartido.

## Lo que NO hace

- **No sustituye a la autenticación** — controla qué filas, no quién eres.
- **El filtro del ORM se puede saltar sin querer** — el SQL crudo (`FromSqlRaw`, Dapper, vistas) no pasa por los query filters; ahí RLS es imprescindible.
- **No protege contra un `TenantId` mal asignado** — si guardas una fila con el tenant equivocado, el filtro la "esconderá" del dueño real; el problema está en la escritura, no en la lectura.
- **RLS no es gratis** — añade una pequeña sobrecarga y requiere configurar la variable de sesión en cada conexión.

## Buenas prácticas avanzadas

- **Claves foráneas compuestas para impedir referencias cruzadas** — el query filter evita *leer* filas ajenas, pero no impide que un pedido de Acme apunte a un cliente de Globex vía `CustomerId`. Si la FK incluye el tenant, la propia base lo hace imposible:

  ```sql
  ALTER TABLE Orders ADD CONSTRAINT FK_Orders_Customers
      FOREIGN KEY (TenantId, CustomerId) REFERENCES Customers (TenantId, Id);
  ```

- **Haz que el filtro "falle cerrado"** — si el tenant context está vacío (un job en segundo plano, un test mal montado), el filtro no debe degradarse a "sin filtro": debe devolver cero filas o lanzar una excepción. Un `_tenantContext.Current?.Id` que se evalúa a `null` y deja pasar todo es la fuga más silenciosa que existe; que la ausencia de tenant duela pronto y a gritos.
- **Trata `IgnoreQueryFilters()` como material radiactivo** — es la puerta de atrás oficial del ORM y siempre acaba usándose "solo para este informe". Limita su uso a un módulo administrativo concreto y añade un chequeo en la CI (un simple *grep* del repositorio) que obligue a justificar cada aparición nueva en revisión de código.
- **Con RLS, bloquea también la escritura** — un `FILTER PREDICATE` (como el del ejemplo) solo oculta filas al *leer*; sin un `BLOCK PREDICATE`, una consulta con bug aún puede insertar o actualizar filas con el `TenantId` de otro cliente. La política completa lleva ambos; es el detalle que casi todo el mundo descubre tarde.
- **`TenantId` primero en los índices** — todas tus consultas llevan el filtro por tenant, explícito o inyectado; si los índices no empiezan por `TenantId`, el motor recorre filas de todos los clientes para quedarse con las de uno. Es el motivo nº 1 de "la app iba bien con 10 tenants y se arrastra con 500".

---

*En resumen: aislar por fila es marcar cada dato con su dueño y poner un guardia automático —en el ORM y en la base de datos— que tape lo que no es tuyo, para que el aislamiento no dependa de recordar un `WHERE`.*
