# Blazor

## ¿Qué es?

Blazor es la tecnología de ASP.NET Core que te permite construir interfaces web
interactivas escribiendo **C# en lugar de JavaScript**. La UI se organiza en
**componentes** que combinan marcado (HTML) y lógica en C#, con enlace de datos
(*data binding*) parecido al de WPF.

## ¿Por qué existe?

Es el puente más directo entre WPF y la web. En WPF tenías componentes (controles y
`UserControl`), enlace de datos bidireccional y manejadores de eventos en C#. Blazor te
da exactamente eso, pero para el navegador: componentes reutilizables, binding y eventos,
todo en C#, sin tener que aprender JavaScript ni un framework de frontend para empezar a
ser productiva.

> Si vienes de WPF, Blazor es lo más parecido a casa: un componente `.razor` es como un
> `UserControl` (marcado + código), `@bind` es tu binding bidireccional y `@onclick`
> es tu manejador de evento. La diferencia es que el resultado se dibuja en un navegador.

## ¿Cuándo y para qué se usa?

Cuando quieres una interfaz web rica e interactiva sin salir del ecosistema .NET: paneles
internos, formularios complejos, dashboards, herramientas de gestión. Por ejemplo, un
formulario de alta de pedido que valida y actualiza totales al instante mientras la
usuaria escribe —algo que en una web clásica exigiría JavaScript— en Blazor lo escribes
en C#.

**Dos modos de ejecución** (la decisión clave al empezar):

- **Blazor Server**: el C# corre en el servidor y la pantalla se actualiza por una
  conexión en tiempo real. Arranca rápido y es ligero para el navegador, pero necesita
  conexión constante con el servidor.
- **Blazor WebAssembly (WASM)**: tu C# se descarga y corre **dentro del navegador**
  gracias a WebAssembly. Funciona como una app de escritorio cargada en la web e incluso
  sin conexión, a costa de una descarga inicial mayor.

## Lo mínimo que necesitas saber

**1. Un componente mezcla marcado y C# en un archivo `.razor`**

```razor
<h3>Contador</h3>
<p>Llevas @cuenta clics</p>
<button @onclick="Incrementar">Súmame uno</button>

@code {
    private int cuenta = 0;
    private void Incrementar() => cuenta++; // la UI se actualiza sola
}
```

> Fíjate: `@onclick="Incrementar"` es el equivalente directo a enganchar el `Click` de un
> `Button` a un método en WPF.

**2. El enlace de datos bidireccional se hace con `@bind`**

Igual que el `{Binding ... Mode=TwoWay}` de XAML, `@bind` mantiene sincronizados el campo
y lo que se ve en pantalla:

```razor
<input @bind="nombre" />
<p>Hola, @nombre</p>

@code {
    private string nombre = "";
}
```

**3. Los componentes se anidan y reciben datos por parámetros**

Como anidabas `UserControl`s en WPF, aquí anidas componentes y les pasas datos con
parámetros:

```razor
<TarjetaProducto Producto="miProducto" />
```

```razor
@* TarjetaProducto.razor *@
<div class="tarjeta">@Producto.Nombre — @Producto.Precio €</div>

@code {
    [Parameter] public Producto Producto { get; set; } = default!;
}
```

**4. Usa la misma inyección de dependencias que el resto de ASP.NET Core**

Pides tus servicios con `@inject` y los usas como en cualquier clase:

```razor
@inject IProductoRepository Repo

@code {
    private List<Producto> productos = new();
    protected override async Task OnInitializedAsync()
        => productos = await Repo.ObtenerTodosAsync(); // como un Loaded async
}
```

**5. Sigue siendo web por debajo**

Aunque programes en C#, lo que se dibuja es HTML y CSS, y por debajo hay HTTP. Conocer
los fundamentos web ([HTML/CSS](HTML-y-CSS.md), [HTTP](HTTP.md)) te ayudará cuando algo
no encaje, aunque no escribas JavaScript a diario.

## Lo que NO hace

- **No elimina la web** — sigues necesitando entender HTML, CSS y el modelo cliente-servidor por debajo.
- **No es siempre la mejor opción** — para sitios muy públicos o SEO-críticos, a veces conviene HTML renderizado en servidor o un framework JS.
- **Blazor Server no funciona sin conexión** — depende de una conexión viva con el servidor; si la quieres offline, ese es el caso de WebAssembly.

## Buenas prácticas avanzadas

- **Usa `@key` cuando pintes listas que cambian** — al renderizar una lista con `@foreach`, Blazor reutiliza los componentes por posición: si insertas o reordenas elementos, el estado interno de un componente puede acabar "pegado" al producto equivocado (el clásico input que conserva el texto de otra fila). Añadir `<TarjetaProducto @key="producto.Id" ... />` le dice a Blazor qué componente corresponde a qué dato, como hacía el contenedor de items de WPF con sus `DataTemplate`.
- **Cuenta con que `OnInitializedAsync` puede ejecutarse dos veces** — con el *prerendering* activado (el modo por defecto en las plantillas modernas), el componente se renderiza primero en el servidor para enviar HTML rápido y luego otra vez al volverse interactivo: tu carga de datos se dispara dos veces. Si eso duele (llamadas caras, efectos secundarios), cachea el resultado entre ambos renders con `PersistentComponentState` o desactiva el prerender para ese componente.
- **Desde un hilo de fondo, la UI se toca con `InvokeAsync`** — los eventos de UI (`@onclick`, `@bind`) ya re-renderizan solos, pero si el cambio viene de un `Timer`, un evento de SignalR o cualquier hilo externo, hay que envolverlo: `await InvokeAsync(StateHasChanged)`. Es el equivalente exacto al `Dispatcher.Invoke` de WPF, y olvidarlo produce el mismo síntoma: la pantalla no se entera del cambio (o Blazor lanza una excepción de hilo).
- **Los componentes también tienen fugas: implementa `IDisposable`** — un componente que se suscribe a un evento de un servicio singleton o arranca un `Timer` y no se desuscribe queda retenido en memoria aunque la usuaria navegue a otra página. En Blazor Server, multiplicado por cada circuito abierto, tumba el servidor con el tiempo. Es la misma fuga por eventos de WPF, con el agravante de que ahora hay miles de "ventanas" a la vez: `@implements IDisposable` y limpia en `Dispose`.

---

*En resumen: Blazor te deja construir interfaces web interactivas con componentes, binding y eventos en C# —el puente más natural desde WPF— evitando tener que aprender JavaScript para empezar.*
