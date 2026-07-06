# Multi-Tenancy — Guía de tecnología

Guía introductoria (pero con algo más de profundidad) sobre **multi-tenancy**: cómo una sola aplicación da servicio a muchos clientes —*tenants*— manteniendo sus datos separados. Está escrita para perfiles junior con nociones básicas de programación y algo de backend, sin asumir experiencia previa con SaaS.

Cada ficha explica un concepto: qué es, por qué existe, cuándo se usa y lo mínimo que necesitas saber para no perderte. Los ejemplos son genéricos (una tienda online, un CRM, una app de facturación) y usan C#/.NET, pero las ideas son aplicables a cualquier stack.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Cada concepto apoya al siguiente: primero la idea general, luego dónde guardar los datos, cómo saber de quién es cada petición y, por último, los retos de hacerlo a escala.

### 1. El concepto

Empieza aquí para entender de qué va todo.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Multi-Tenancy](Multi-Tenancy.md) | La idea base: una app, muchos clientes, datos separados. Sin esto, el resto no encaja. |

### 2. Dónde viven los datos

El corazón del diseño: cómo se separan físicamente los datos de cada tenant.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [Estrategias de Aislamiento](Estrategias-de-Aislamiento.md) | Base por tenant, esquema por tenant o base compartida: el equilibrio entre aislamiento, coste y complejidad. |

### 3. Cómo sabe la app de quién es cada petición

Antes de leer o escribir nada, la aplicación debe identificar el tenant y tenerlo a mano.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [Identificación del Tenant](Identificacion-del-Tenant.md) | *Tenant resolution*: averiguar a qué tenant pertenece cada petición (subdominio, cabecera, token...). |
| 4 | [Tenant Context](Tenant-Context.md) | Dónde se guarda el tenant actual para que todas las capas lo usen sin pasarlo a mano. |

### 4. Garantizar que nadie vea datos de otro

La pieza de seguridad central cuando se comparte la base de datos.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 5 | [Aislamiento por Fila](Aislamiento-por-Fila.md) | Filtrar siempre por `TenantId` de forma automática (query filters y Row-Level Security). |

### 5. Operar a escala

Lo que aparece cuando el producto crece: dar de alta clientes, migrar a todos y convivir bien.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 6 | [Aprovisionamiento y Migraciones](Aprovisionamiento-y-Migraciones.md) | Crear el espacio de cada tenant nuevo y aplicar cambios de esquema a todos. |
| 7 | [Retos Avanzados](Retos-Avanzados.md) | Vecino ruidoso, configuración por tenant, caché y fugas entre tenants. |

---

> ¿Te interesa cómo se organiza el código que sostiene todo esto? Echa un vistazo a la guía de [Clean Architecture](../clean-architecture/README.md) para ver dónde encajarían el tenant context y los repositorios.
