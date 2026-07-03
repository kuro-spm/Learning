# Módulos vs Shared

## ¿Qué es?

En un **monolito modular** el código se organiza en dos tipos de "casa": los **módulos** (cada capacidad de negocio con sus propios proyectos de Application e Infrastructure) y los **proyectos compartidos** o *shared* (piezas comunes que varios módulos reutilizan). Esta ficha responde a una pregunta que aparece a diario: *"tengo código nuevo, ¿va a un módulo o va a shared?"*.

## ¿Por qué existe?

Cuando una aplicación crece, meterlo todo en un mismo montón produce el famoso "código espagueti": cualquier cambio toca veinte sitios. La solución habitual es partir la aplicación en **módulos**, cada uno dueño de un trozo del negocio. Pero en cuanto existen dos módulos aparece la tentación opuesta: "esto lo pueden necesitar otros… lo pongo en shared, por si acaso". Si cedes a esa tentación con todo, shared engorda hasta convertirse en un cajón de sastre del que dependen todos los módulos, y vuelves al punto de partida: cualquier cambio afecta a todo.

Por eso hace falta un **criterio de reparto** claro. Sin él, la arquitectura modular es solo carpetas bonitas.

> Piensa en un edificio de apartamentos. Cada módulo es un apartamento: tiene sus muebles, su cocina y su cerradura, y sus dueños lo reforman cuando quieren sin pedir permiso a nadie. Shared son las zonas comunes: la escalera, el portal, las tuberías generales. Nadie mete su sofá en el portal "por si otro vecino lo quiere usar", y nadie reforma la escalera cada semana, porque un cambio ahí afecta a todo el edificio.

## ¿Cuándo y para qué se usa?

Esta decisión aparece cada vez que creas una clase nueva en un monolito modular: una entidad, un caso de uso, un repositorio, un cliente HTTP, un tipo de respuesta. Para decidir bien, primero hay que tener claro qué es cada cosa:

**Un módulo** es un *bounded context*: una capacidad de negocio completa, con sus **casos de uso**, sus **tablas** y su **ciclo de vida propio** (evoluciona a su ritmo, con sus propios requisitos). Se materializa como una pareja de proyectos: `Modulo.Application` (lógica y casos de uso) y `Modulo.Infrastructure` (repositorios, migraciones, clientes externos). En una tienda online: Catálogo, Pedidos, Envíos, Autenticación… cada uno es un módulo.

**Los proyectos shared** son tres, cada uno con un papel concreto:

- **SharedKernel** — tipos base de **dominio** sin dependencias: entidades e interfaces que varios módulos necesitan conocer, tipos base como `IBaseEntity`. Es la capa más estable de toda la aplicación.
- **SharedApplication** — **contratos entre módulos** y tipos de respuesta comunes (por ejemplo, un wrapper de resultado tipo `BaseCommandResponse`): lo que un módulo expone para que otros le hablen sin conocer sus tripas.
- **SharedInfrastructure** — infraestructura **técnica** común: factoría de conexiones a base de datos, runner de migraciones, utilidades de logging. Cosas de fontanería, sin una gota de negocio.

**El criterio principal**, en una frase: *shared guarda lo estable y genérico que necesitan **dos o más** módulos; los módulos guardan lo que **cambia con el negocio***. De ahí sale la **regla del 2**: nada se mueve a shared por anticipación ("seguro que alguien lo usará"); se mueve cuando el segundo consumidor real aparece. Hasta entonces, vive en su módulo.

## Lo mínimo que necesitas saber

**1. El ejemplo guía: la autenticación es un módulo, aunque "todos la usen"**

Es el error más común. En una tienda online, la autenticación tiene casos de uso propios (login, logout), tablas propias (usuarios, sesiones) y evoluciona con requisitos de negocio (caducidad de sesión, bloqueo de cuentas, doble factor…). Eso es exactamente la definición de módulo. Que muchos otros módulos la necesiten no la convierte en shared: lo que va a shared no es el módulo, sino sus **contratos**.

```csharp
// Módulo Auth.Application → el caso de uso (cambia con el negocio)
public class LoginManager
{
    public async Task<BaseCommandResponse> Login(string email, string password) { ... }
}

// SharedKernel → la entidad y la interfaz que OTROS módulos necesitan conocer
public class Usuario : IBaseEntity { ... }
public interface IUsuarioRepository { Task<Usuario> GetByEmail(string email); }

// SharedApplication → el tipo de respuesta común a todos los módulos
public class BaseCommandResponse { public bool Success; public string Error; }
```

**2. Los módulos se hablan por contratos, nunca en directo**

Si el módulo Pedidos necesita datos de un usuario, no referencia el proyecto `Auth.Application`: usa el contrato compartido (`IUsuarioRepository` en SharedKernel). Así Auth puede reformar sus tripas sin romper a Pedidos.

```csharp
// ✅ Pedidos depende del contrato compartido
public class CrearPedidoManager(IUsuarioRepository usuarios) { ... }

// ❌ Pedidos NO referencia el proyecto de otro módulo
// using Auth.Application;  ← prohibido: acopla los dos módulos para siempre
```

**3. La regla del 2: no muevas nada a shared por anticipación**

Un tipo nace en su módulo. Solo asciende a shared cuando un **segundo módulo real** lo necesita — no cuando "quizá algún día" alguien lo use. Mover algo a shared es fácil; sacarlo de shared cuando ya dependen tres módulos es una obra.

```text
Cliente HTTP de la pasarela de pago, usado solo por Pedidos
  → vive en Pedidos.Infrastructure
Mañana Suscripciones también cobra con esa pasarela (2.º consumidor real)
  → AHORA sí: se valora subir el contrato a shared
```

**4. Tabla de decisión rápida**

| Tipo de código | Destino |
|---|---|
| Entidad de un dominio concreto (`Pedido`, `Usuario`) | Su módulo — o SharedKernel si otros módulos la consumen vía interfaz |
| Tipo base genérico (`IBaseEntity`, wrapper de resultado) | Shared (SharedKernel / SharedApplication) |
| Caso de uso, Manager, Strategy | **Siempre** su módulo — la lógica de negocio nunca es shared |
| Cliente HTTP de un servicio externo usado por un solo módulo | Ese módulo |
| Factoría de conexión a BD, runner de migraciones | SharedInfrastructure |

**5. Señales de alarma en ambos sentidos**

- **Shared que engorda con lógica de negocio.** Si en shared aparecen `if` de negocio, Managers o casos de uso, se está convirtiendo en un cajón de sastre que acopla todo: cualquier cambio obliga a recompilar (y repensar) todos los módulos a la vez.
- **Módulos que se referencian entre sí directamente.** Si `Pedidos` tiene un `using` de `Auth.Application`, ya no tienes dos módulos: tienes uno grande disfrazado. La comunicación va siempre por contratos compartidos.

## Lo que NO hace

- **Crear un módulo no aísla nada por sí solo** — si metes sus tipos de dominio en shared indiscriminadamente, el aislamiento es de mentira: todos siguen acoplados a todo a través de shared.
- **Shared no es un atajo para saltarse la dirección de dependencias** — sigue aplicando la regla de Clean Architecture: SharedKernel no depende de nadie, y la infraestructura nunca se cuela en los contratos.
- **La regla del 2 no prohíbe compartir** — solo pide que el segundo consumidor sea real, no imaginario. Cuando aparece, mover el contrato a shared es la jugada correcta.
- **Esta división no decide el despliegue** — módulos y shared conviven en un único ejecutable (por eso es un *monolito* modular); es una frontera de código, no de servidores.

---

*En resumen: los módulos guardan lo que cambia con el negocio (casos de uso, tablas, ciclo de vida propio) y shared guarda solo lo estable y genérico que ya necesitan dos o más módulos — cuando dudes, déjalo en su módulo, que ascender a shared siempre estás a tiempo.*
