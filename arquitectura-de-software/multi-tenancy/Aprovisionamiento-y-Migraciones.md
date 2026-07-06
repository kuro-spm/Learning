# Aprovisionamiento y Migraciones

## ¿Qué es?

El **aprovisionamiento** (*provisioning*) es todo lo que ocurre cuando nace un tenant nuevo: crear su espacio de datos, su configuración inicial y dejarlo listo para usarse. Las **migraciones** son los cambios de estructura de la base de datos (nuevas tablas, columnas...) que hay que aplicar a *todos* los tenants a medida que el producto evoluciona.

## ¿Por qué existe?

En una app de un solo cliente, montas la base de datos una vez y migras una vez. En multi-tenancy esto se multiplica: cada nuevo cliente necesita su rincón preparado, y cada cambio del modelo de datos hay que llevarlo a cientos o miles de tenants sin que ninguno se quede atrás ni se rompa.

> Piensa en una cadena de tiendas. Abrir una tienda nueva (provisioning) implica montar estanterías, caja y catálogo inicial. Y cuando central decide cambiar el sistema de cajas (migración), hay que actualizar **todas** las tiendas, no solo una, y sin cerrar el negocio.

Cuanto más aislada sea tu estrategia (base por tenant), más trabajo de provisioning y migración por cliente; cuanto más compartida, más sencillo. Es la otra cara del equilibrio que vimos en [Estrategias de Aislamiento](Estrategias-de-Aislamiento.md).

## ¿Cuándo y para qué se usa?

El provisioning se dispara al dar de alta un cliente (un registro automático, o un proceso comercial). Las migraciones se aplican en cada despliegue en el que cambie el esquema. En modelos serios, ambos son procesos automatizados, no pasos manuales.

## Lo mínimo que necesitas saber

**1. Provisioning: registrar el tenant y crear su espacio**

Al alta, se crea el registro del tenant y se prepara su almacenamiento (su base/esquema, o simplemente sus filas de configuración si la base es compartida).

```csharp
public async Task<Tenant> ProvisionAsync(string name, string subdomain)
{
    var tenant = new Tenant { Id = Guid.NewGuid(), Name = name, Subdomain = subdomain, IsActive = true };
    await _tenantStore.AddAsync(tenant);

    await CreateStorageAsync(tenant);   // base/esquema dedicado, o nada si es compartida
    await SeedDefaultsAsync(tenant);    // roles, configuración y datos iniciales
    return tenant;
}
```

**2. Datos semilla (*seeding*) iniciales**

Un tenant recién creado suele necesitar un mínimo para funcionar: un usuario administrador, roles por defecto, ajustes base. Sin esto, el cliente entra a una app vacía e inservible.

```csharp
await _roles.CreateAsync(tenant.Id, "Administrador");
await _settings.SetAsync(tenant.Id, "currency", "EUR");
```

**3. Migraciones en base compartida: una vez para todos**

Si todos comparten tablas, una sola migración actualiza a todos los tenants de golpe. Es lo más cómodo.

```bash
dotnet ef database update
```

**4. Migraciones en base por tenant: recorrer todos**

Si cada tenant tiene su base, hay que aplicar la migración a cada una. Se automatiza recorriendo la lista de tenants.

```csharp
foreach (var tenant in await _tenantStore.GetAllActiveAsync())
{
    var options = BuildOptionsFor(tenant);     // su cadena de conexión
    using var db = new AppDbContext(options);
    await db.Database.MigrateAsync();          // migra esa base concreta
}
```

**5. Tener en cuenta el medio camino**

Migrar miles de bases lleva tiempo y alguna puede fallar. Conviene registrar qué tenant ya está migrado, poder reintentar los que fallen y evitar que el código nuevo asuma un esquema que aún no se ha aplicado en todos.

## Lo que NO hace

- **El provisioning no es solo "crear la base"** — incluye los datos semilla; sin ellos el tenant nace inservible.
- **Las migraciones por tenant no son atómicas entre sí** — puede quedar la mitad migrada; necesitas seguimiento y reintentos.
- **Borrar un tenant no es borrar un registro** — hay que eliminar (o archivar) todos sus datos, respetando obligaciones legales de retención.
- **No improvises en producción** — provisioning y migraciones deben ser procesos repetibles y automatizados, no comandos manuales sueltos.

## Buenas prácticas avanzadas

- **Provisioning idempotente y con estado explícito** — un alta que falla a mitad (base creada, semillas no) deja un tenant zombi que ni funciona ni se puede repetir. Diseña cada paso para poder reejecutarse sin duplicar nada (las semillas como *upsert*, la creación como "crear si no existe") y modela el estado del tenant (`Provisioning` → `Active` → `Suspended`): la aplicación solo deja entrar cuando está `Active`, y un alta fallida simplemente se relanza.
- **Migra con el patrón *expand/contract*** — con muchos tenants, la migración no es instantánea: durante horas convivirán código nuevo y esquema viejo (o al revés). Nunca hagas un cambio destructivo en un paso; primero *expande* (añade la columna nueva, tolerada por el código viejo), despliega código que funcione con ambas versiones, rellena datos, y solo al final *contrae* (elimina lo antiguo). Renombrar una columna "de golpe" funciona en local y rompe en producción multi-tenant.
- **Ensaya cada migración con tenants canario** — antes de recorrer miles de bases, aplica la migración a los tenants internos o de demo, después a un porcentaje pequeño de clientes reales, y solo entonces al resto. Una migración que corrompe datos aplicada a *todos* los tenants a la vez es el peor día posible de un SaaS; aplicada a tres canarios, es un susto.
- **Paraleliza el recorrido, pero con techo y con registro por tenant** — migrar en serie miles de bases tarda una eternidad; en paralelo sin límite, tumba el servidor de base de datos. Procesa por lotes con un grado de paralelismo fijo y guarda la versión de esquema alcanzada por cada tenant: eso convierte "¿por dónde iba?" tras un fallo en una consulta, y los reintentos en algo trivial.

---

*En resumen: aprovisionar es montarle la tienda a cada cliente nuevo y migrar es reformar todas las tiendas a la vez sin cerrar —cuanto más aislada la estrategia, más trabajo por cliente.*
