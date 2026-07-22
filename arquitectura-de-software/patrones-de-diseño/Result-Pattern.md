# Result Pattern

## ¿Qué es?

El Result Pattern (también llamado *Operation Result*) consiste en devolver explícitamente el éxito o el fracaso de una operación como parte del valor de retorno de un método, en vez de usar excepciones para comunicar los fallos que la aplicación ya espera que puedan ocurrir.

## ¿Por qué existe?

Una excepción está pensada para lo **excepcional**: un fallo que nadie esperaba, como perder la conexión a la base de datos. Usarla también para fallos habituales del negocio —un email ya registrado, un pedido que no existe, un descuento no aplicable— tiene dos problemas. Primero, lanzar y capturar excepciones tiene un coste de rendimiento notable frente a un simple `return`. Segundo, y más importante, una excepción no aparece en la firma del método: quien lo llama no tiene forma de saber, solo mirando el tipo de retorno, que esa operación puede fallar y de qué maneras.

El Result Pattern hace visible el fallo en el propio tipo de retorno. El compilador (o al menos el propio código) obliga a comprobar si la operación fue bien antes de usar su resultado, de la misma forma que una función SQL que no encuentra filas no lanza un error, simplemente devuelve un conjunto vacío que hay que comprobar.

> Si conoces los códigos de estado HTTP: un `404 Not Found` es un Result Pattern aplicado a nivel de API — la petición no lanza una excepción al cliente, le devuelve un valor (el código) que el cliente está obligado a mirar antes de interpretar la respuesta.

## ¿Cuándo y para qué se usa?

Cuando un método puede fallar de una forma que **forma parte del flujo normal** de la aplicación y que quien lo llama debe manejar sí o sí: una validación que no pasa, una regla de negocio que se incumple, un recurso que no se encuentra, un conflicto de estado. No sustituye a las excepciones para fallos verdaderamente inesperados (una conexión caída, un bug, una invariante rota): esos casos siguen siendo mejor comunicados con una excepción, porque nadie diseña código pensando en manejarlos como parte del camino feliz.

## Lo mínimo que necesitas saber

**1. Un `Result` básico distingue éxito de fracaso**

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }

    private Result(bool isSuccess, T? value, string? error)
        => (IsSuccess, Value, Error) = (isSuccess, value, error);

    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);
}
```

**2. El método devuelve el resultado, no lanza**

```csharp
public async Task<Result<Pedido>> ConfirmarPedido(int pedidoId)
{
    var pedido = await repositorio.ObtenerPorId(pedidoId);
    if (pedido is null)
        return Result<Pedido>.Failure("El pedido no existe.");

    pedido.Confirmar();
    await repositorio.Guardar(pedido);
    return Result<Pedido>.Success(pedido);
}
```

**3. Quien llama está obligado a comprobar el resultado antes de usar el valor**

```csharp
var resultado = await confirmarPedido.Ejecutar(pedidoId);
if (!resultado.IsSuccess)
    return BadRequest(resultado.Error);

return Ok(resultado.Value);
```

**4. Un código de motivo, en vez de solo texto, permite decidir sin parsear strings**

```csharp
public enum MotivoFallo { NoEncontrado, Validacion, Conflicto }

public class Result<TData, TMotivo>
{
    public bool IsSuccess { get; init; }
    public TData? Data { get; init; }
    public TMotivo? Motivo { get; init; }
    public string? Error { get; init; }
}

// En el controlador, el motivo decide el código HTTP sin comparar cadenas de texto
return resultado.Motivo switch
{
    MotivoFallo.NoEncontrado => NotFound(resultado.Error),
    MotivoFallo.Validacion => BadRequest(resultado.Error),
    _ => Conflict(resultado.Error)
};
```

## Lo que NO hace

- **No elimina las excepciones** — para fallos verdaderamente inesperados (una dependencia caída, una invariante rota, un bug) seguir lanzando una excepción sigue siendo correcto; el Result Pattern es solo para fallos que la aplicación ya contempla como parte de su lógica.
- **No es gratis** — cada método que lo adopta obliga a todos sus llamadores a comprobar el resultado explícitamente; si alguien ignora esa comprobación, el fallo pasa desapercibido tan silenciosamente como un `catch` vacío.
- **No sustituye a la validación de entrada** — a menudo se combina con ella (una validación fallida puede terminar como un `Result` de fracaso), pero son dos cosas distintas: una valida la forma de los datos, la otra comunica el resultado de una operación (ver [FluentValidation](../../desarrollo-web/de-wpf-a-web/FluentValidation.md)).

## Buenas prácticas avanzadas

- **Modela el motivo del fallo como dato, no solo como texto** — un enum o un código permite a quien recibe el resultado (típicamente un controlador) decidir el código HTTP o el flujo a seguir sin comparar cadenas de texto frágiles que se rompen si alguien cambia la redacción de un mensaje.
- **No mezcles excepciones y `Result` para el mismo tipo de fallo** — si un método a veces lanza y a veces devuelve un `Result` fallido para representar el mismo problema, quien lo llama tiene que protegerse de las dos formas a la vez, lo que anula buena parte de la ventaja de hacerlo explícito.
- **El `Result` vive en Application y Domain; el borde lo traduce a protocolo** — el caso de uso devuelve un `Result`, pero es el controlador (o el adaptador que corresponda) quien lo convierte en un código HTTP, un mensaje de cola con estado, o lo que use esa frontera concreta. El caso de uso no debería saber que existe HTTP.
- **Evita anidar comprobaciones en cascada** — encadenar `if (!resultado.IsSuccess) return ...` varias veces seguidas reproduce el mismo problema visual que un `try/catch` anidado. Librerías como FluentResults o CSharpFunctionalExtensions ofrecen operadores (`Bind`, `Map`) para encadenar pasos que pueden fallar sin pirámides de condicionales, aunque un `Result` casero con comprobaciones explícitas también es una opción perfectamente válida si el equipo prefiere no añadir una dependencia más.

---

*En resumen: el Result Pattern convierte un fallo esperado del negocio en un dato que viaja en el propio tipo de retorno, en vez de en una excepción — así quien llama sabe, solo leyendo la firma, que la operación puede no salir bien y está obligado a comprobarlo.*
