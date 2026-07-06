# Clean Architecture

## ¿Qué es?

Clean Architecture es una forma de organizar el código de una aplicación en **capas concéntricas**, donde las reglas de negocio quedan en el centro, aisladas de los detalles técnicos (base de datos, framework web, interfaz...). El objetivo es que lo importante de tu aplicación no dependa de las herramientas con las que está construida.

## ¿Por qué existe?

Cuando un proyecto crece, es muy fácil que todo acabe mezclado: la lógica de negocio metida dentro de un controlador web, las consultas SQL escritas a mano en medio del cálculo de un precio, la regla de "un cliente no puede pedir más stock del disponible" repartida en cinco sitios. El resultado es un código difícil de probar y aterrorífico de cambiar: tocas la base de datos y se rompe la pantalla.

Clean Architecture propone lo contrario: separar **qué hace** tu aplicación (las reglas de negocio) de **cómo lo hace** (la tecnología concreta). Así puedes cambiar de base de datos, de framework o de interfaz sin reescribir la lógica.

> Si ya conoces el patrón Modelo-Vista-Controlador (MVC), piensa en Clean Architecture como "MVC llevado al extremo": no solo separas la vista de la lógica, sino que aíslas la lógica de **absolutamente todo** lo que sea un detalle técnico.

## ¿Cuándo y para qué se usa?

Aparece sobre todo en aplicaciones de negocio que van a vivir y crecer durante años: el backend de una tienda online, la gestión de pedidos de un almacén, una plataforma de reservas. En todos estos casos las reglas ("un pedido pagado no se puede cancelar", "un descuento no puede superar el 50 %") son lo más valioso y lo que más conviene proteger de los cambios técnicos.

No es gratis: añade capas, interfaces y código extra. Para un script pequeño o un prototipo de fin de semana es exagerado. Brilla cuando la lógica de negocio es compleja y el proyecto durará en el tiempo.

## Lo mínimo que necesitas saber

**1. Las capas, de dentro a fuera**

De centro a borde, las responsabilidades van de "puras reglas de negocio" a "puros detalles técnicos":

```text
[ Dominio ]        → entidades y reglas de negocio (lo más estable)
[ Casos de uso ]   → orquestan las acciones de la aplicación
[ Adaptadores ]    → traducen entre el mundo exterior y la aplicación
[ Infraestructura ]→ base de datos, web, frameworks (lo más cambiante)
```

**2. La regla de oro: las dependencias apuntan hacia dentro**

El código de fuera puede conocer al de dentro, pero nunca al revés. El dominio no sabe que existe una base de datos.

```csharp
// ✅ Un caso de uso conoce la entidad del dominio
public class CrearPedido { /* usa la entidad Pedido */ }

// ❌ La entidad Pedido NO conoce al repositorio ni a la base de datos
public class Pedido { /* nada de SqlConnection aquí dentro */ }
```

**3. El dominio define interfaces; la infraestructura las implementa**

El centro dice *qué necesita* (un contrato); el borde decide *cómo* hacerlo.

```csharp
// En el centro: solo el contrato
public interface IRepositorioPedidos
{
    Task Guardar(Pedido pedido);
}

// En el borde: la implementación concreta con la tecnología real
public class RepositorioPedidosSql : IRepositorioPedidos
{
    public Task Guardar(Pedido pedido) { /* SQL aquí */ }
}
```

**4. Lo de fuera se puede sustituir sin tocar el centro**

Como el caso de uso depende de `IRepositorioPedidos` y no de SQL, puedes cambiar a otra base de datos —o usar uno falso en los tests— sin tocar la lógica de negocio.

## Lo que NO hace

- **No es una librería ni un framework** — es un conjunto de principios de organización; no se instala, se aplica.
- **No te dice qué carpetas crear** exactamente — da la idea de las capas, pero la estructura concreta la decides tú.
- **No sustituye al buen diseño** — puedes seguir la estructura al pie de la letra y aun así escribir mala lógica dentro de cada capa.
- **No es siempre la mejor opción** — en proyectos pequeños su sobrecarga no compensa.

## Buenas prácticas avanzadas

- **Organiza las carpetas por negocio, no por tipo técnico** — una estructura que solo muestra `Controllers/`, `Services/` y `Repositories/` no dice nada de qué hace la aplicación. Quien domina la arquitectura agrupa por capacidad de negocio (`Pedidos/`, `Catalogo/`...) y aplica las capas dentro de cada una: al abrir el repositorio se ve *de qué va* el sistema, no con qué framework está hecho (es la idea de la *screaming architecture*).
- **Vigila los paquetes, no solo los `using`** — el acoplamiento más sutil entra por NuGet/npm: basta con que el proyecto de dominio referencie el paquete del ORM para "decorar" una entidad con `[Table]` y el centro ya depende de la base de datos. El proyecto del dominio debería tener cero referencias a paquetes de infraestructura; idealmente, cero referencias externas.
- **No repartas un CRUD en cuatro capas por disciplina** — pasar un dato por entidad + caso de uso + DTO + mapeo cuando no hay ninguna regla de negocio es coste sin beneficio. Los expertos aplican la separación donde hay lógica que proteger y dejan atajos deliberados (y documentados) donde solo hay lectura: por ejemplo, consultas que van directas a la base de datos para pintar un listado, sin pasar por el dominio.
- **Acepta el mapeo entre capas como un coste, no lo "optimices" reutilizando modelos** — usar la misma clase como entidad de dominio, fila de base de datos y respuesta JSON ahorra mapeos hoy y suelda las capas para siempre: renombrar una columna pasa a romper el contrato público de la API. Duplicar DTOs en las fronteras es el precio de poder cambiar cada capa por separado, y es barato comparado con la alternativa.

---

*En resumen: Clean Architecture pone las reglas de negocio en el centro y empuja todos los detalles técnicos al borde, para que lo importante de tu aplicación no dependa de las herramientas con las que está hecha.*
