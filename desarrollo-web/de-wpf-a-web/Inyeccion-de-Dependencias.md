# Inyección de dependencias

## ¿Qué es?

La inyección de dependencias (DI, por sus siglas en inglés) es una forma de organizar el
código en la que una clase **no crea** los objetos que necesita, sino que se los pasan
desde fuera (normalmente por el constructor). En ASP.NET Core hay un sistema de DI
integrado que se encarga de fabricar y entregar esos objetos por ti.

## ¿Por qué existe?

En WPF era muy común que un `ViewModel` creara directamente sus dependencias
(`var repo = new ProductoRepository();`) o tirara de un *singleton* global. Eso ata las
clases entre sí: cambiar una implementación o testear de forma aislada se vuelve difícil.

La DI le da la vuelta: tu clase declara *qué* necesita (una interfaz), y otro se encarga
de *darle* la implementación concreta. ASP.NET Core lleva este patrón integrado de serie
porque encaja de forma natural con su ciclo de vida de peticiones.

> Si en WPF usaste un contenedor de IoC (como los que traen Prism o el toolkit de MVVM),
> esto te resultará familiar: ASP.NET Core trae uno equivalente ya montado, sin instalar nada.

## ¿Cuándo y para qué se usa?

Se usa para todo lo que sea una "dependencia con lógica": repositorios, servicios de
negocio, clientes de APIs externas, el acceso a la base de datos... Por ejemplo, en una
tienda online, un controlador que muestra productos no crea su repositorio: lo recibe ya
listo. Así puedes cambiar de base de datos, o usar un repositorio falso en los tests, sin
tocar el controlador.

## Lo mínimo que necesitas saber

**1. Registras las dependencias al arrancar**

En `Program.cs` le dices al contenedor: "cuando alguien pida esta interfaz, dale esta
implementación".

```csharp
builder.Services.AddScoped<IProductoRepository, ProductoRepository>();
builder.Services.AddScoped<IPedidoService, PedidoService>();
```

**2. Las pides por el constructor; el framework las inyecta**

No haces `new`: declaras lo que necesitas como parámetro del constructor y ASP.NET Core
te lo entrega ya construido.

```csharp
public class PedidoService : IPedidoService
{
    private readonly IProductoRepository _productos;

    public PedidoService(IProductoRepository productos)
    {
        _productos = productos; // te llega ya creado, no haces new
    }
}
```

**3. Cada dependencia tiene un tiempo de vida**

Eliges cuánto debe vivir cada objeto. Esta es una decisión importante en web, donde hay
muchas peticiones a la vez:

```text
AddSingleton  → una sola instancia para toda la aplicación (cuidado: la comparten todos los clientes)
AddScoped     → una instancia por petición HTTP (lo más habitual para repositorios y servicios)
AddTransient  → una instancia nueva cada vez que se pide
```

> `Scoped` es la opción típica: cada petición de cada cliente recibe sus propios objetos,
> evitando que se mezclen datos entre usuarias.

**4. Encaja con programar contra interfaces**

La DI brilla cuando dependes de interfaces (`IProductoRepository`) en vez de clases
concretas. Así puedes intercambiar la implementación —real, en memoria, falsa para
tests— sin tocar el código que la usa. Es la base de una [arquitectura limpia](../../arquitectura-de-software/clean-architecture/README.md).

## Lo que NO hace

- **No adivina qué implementación quieres** — si no registras una interfaz, te dará un error al pedirla.
- **No gestiona estado de usuaria** — sirve dependencias, no guarda quién está conectado.
- **No sustituye al diseño** — inyectar bien depende de que hayas separado responsabilidades en interfaces.

## Buenas prácticas avanzadas

- **Vigila la "dependencia cautiva": un singleton no debe recibir un scoped** — si un servicio `Singleton` recibe en su constructor algo registrado como `Scoped` (un `DbContext`, por ejemplo), esa instancia queda atrapada para siempre dentro del singleton y se comparte entre todas las peticiones: datos mezclados entre usuarias y errores de concurrencia intermitentes. En el entorno `Development`, el contenedor valida esto y explota al arrancar (`ValidateScopes`); si ves ese error, no lo "arregles" cambiando el tiempo de vida al azar: entiende quién debe vivir cuánto.
- **Fuera de una petición HTTP no hay *scope*: créalo tú** — en un `BackgroundService` o un trabajo programado no existe la petición que normalmente delimita los `Scoped`. Inyecta `IServiceScopeFactory` y crea el ámbito a mano:

  ```csharp
  using var scope = _scopeFactory.CreateScope();
  var repo = scope.ServiceProvider.GetRequiredService<IProductoRepository>();
  ```

  Sin esto, o no puedes resolver el servicio, o acabas con una dependencia cautiva.
- **No inyectes `IServiceProvider` para ir pidiendo cosas sueltas** — pedir servicios con `GetService<T>()` desde cualquier parte (el antipatrón *Service Locator*) esconde las dependencias reales de la clase: ya no basta mirar el constructor para saber qué necesita, y los tests dejan de avisarte cuando falta algo. El constructor es el contrato; mantenlo honesto.
- **Un constructor con siete dependencias es un aviso, no un trámite** — la DI hace tan fácil añadir "una dependencia más" que las clases engordan sin fricción. Cuando el constructor no cabe en una línea de vista, la clase casi seguro tiene demasiadas responsabilidades; el arreglo es dividirla o agrupar dependencias que siempre van juntas, no acostumbrarse a la lista.

---

*En resumen: la inyección de dependencias hace que tus clases pidan lo que necesitan en lugar de crearlo; ASP.NET Core trae el contenedor integrado y tú solo registras qué implementación entregar y cuánto debe vivir.*
