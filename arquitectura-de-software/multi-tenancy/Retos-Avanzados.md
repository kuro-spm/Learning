# Retos Avanzados

**¿Qué es?** Un repaso a los problemas que aparecen cuando una aplicación multi-tenant crece: clientes que se molestan entre sí por compartir recursos, configuración propia de cada tenant, cachés que deben separar datos y los temidos escapes de información entre tenants. Son los detalles que distinguen un SaaS de juguete de uno de producción.

---

## ¿Por qué existe?

El aislamiento de datos (las fichas anteriores) es el punto de partida, no el final. Cuando compartes una aplicación entre muchos clientes, *todo* lo demás también se comparte: la CPU, la memoria, las cachés, los límites de la API. Y cada cliente quiere sentir que la app es "suya": su logo, sus reglas, sus límites. Estos retos surgen precisamente de ese reparto de un recurso común entre inquilinos con necesidades distintas.

> Vuelve al edificio de apartamentos: una vez resuelto que nadie entre en tu piso, quedan los problemas de convivencia —el vecino ruidoso, la decoración de tu casa, que el ascensor no se sature a las 8:00.

---

## ¿Cuándo y para qué se usa?

Estos temas se vuelven urgentes cuando el producto pasa de unos pocos clientes a muchos, o cuando aparece un cliente grande que consume mucho más que el resto. Conviene tenerlos en el radar desde el diseño, aunque no los resuelvas todos el primer día.

---

## Lo mínimo que necesitas saber

**1. El vecino ruidoso (*noisy neighbor*)**

Un tenant que consume muchos recursos (consultas pesadas, picos de tráfico) puede degradar el servicio para todos los demás. Se mitiga con límites por tenant (*rate limiting*, cuotas) y, para clientes grandes, recursos dedicados.

```csharp
// Límite de peticiones por tenant, no global
rateLimiter.Configure(tenantId, maxRequestsPerMinute: 600);
```

**2. Configuración por tenant**

Cada cliente quiere personalizar: idioma, moneda, logo, qué funciones tiene activas (*feature flags*). Se guarda asociada al `TenantId` y se carga junto con el tenant context.

```csharp
var settings = await _settings.GetForAsync(tenant.Id);
if (settings.Features.Contains("export_pdf")) { /* habilitar exportación */ }
```

**3. La caché también es multi-tenant**

Una caché compartida es una fuente clásica de fugas: si guardas `"orders"` y devuelves lo mismo a todos, mezclas datos. La clave de caché **debe** incluir el tenant.

```csharp
// MAL: var key = "orders";
var key = $"orders:{tenant.Id}";   // BIEN: cada tenant, su entrada
```

**4. Fugas entre tenants (*cross-tenant leakage*)**

Es el fallo más grave: que un tenant acceda a datos de otro. No solo ocurre en consultas; vigila también IDs adivinables en URLs, trabajos en segundo plano que pierden el contexto del tenant, y logs o exportaciones que mezclan clientes.

```csharp
// Un job en background NO tiene request: hay que fijarle el tenant explícitamente
foreach (var tenant in tenants)
    using (tenantContext.BeginScope(tenant))   // sin esto, el job corre "sin tenant"
        await ProcessAsync();
```

**5. Observabilidad por tenant**

Para diagnosticar problemas necesitas saber *qué tenant* los sufre. Etiqueta logs y métricas con el `TenantId` (sin volcar datos sensibles), para poder aislar incidencias por cliente.

---

## Lo que NO hace

- **El aislamiento de datos no resuelve el rendimiento** — un tenant puede no ver datos de otro y aun así ralentizarle; son problemas distintos.
- **La caché no se aísla sola** — si olvidas el tenant en la clave, filtras datos aunque las consultas estén perfectas.
- **Los procesos en segundo plano no heredan el tenant de una petición** — no hay request; debes establecer el contexto a mano.
- **No hay "terminado" definitivo** — escalar multi-tenancy es un ajuste continuo a medida que cambian el número y el tamaño de los clientes.

---

*En resumen: una vez aislados los datos, el verdadero reto es la convivencia —que un cliente no moleste a otro, que cada uno tenga lo suyo y que nada (ni una caché, ni un job, ni un log) se cuele entre tenants.*
