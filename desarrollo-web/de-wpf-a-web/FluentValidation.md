# FluentValidation

## ¿Qué es?

FluentValidation es una librería .NET para validar objetos —normalmente los datos de entrada de una petición— escribiendo las reglas con una sintaxis fluida y encadenable, en una clase separada del propio objeto que se valida.

## ¿Por qué existe?

Validar "a mano" dentro de un controlador (una sucesión de `if (string.IsNullOrEmpty(...)) return BadRequest(...)`) crece sin control y mezcla el manejo de la petición con las reglas de qué es un dato válido. Poner atributos de validación directamente sobre la clase de entrada (`[Required]`, `[MaxLength]`) tampoco escala bien en cuanto una regla depende de otro campo, cambia según el contexto (crear frente a actualizar el mismo tipo de dato), o necesita consultar algo externo.

FluentValidation separa "qué forma tienen los datos" (la clase) de "qué reglas deben cumplir" (el validador), en una clase propia, testeable de forma aislada, con una sintaxis pensada para leerse casi como una frase en lugar de como una maraña de condicionales.

> Si conoces las anotaciones de datos de .NET (`[Required]`, `[Range]`): FluentValidation resuelve el mismo problema, pero como código en vez de como atributos, lo que permite reglas condicionales, mensajes dinámicos y validar contra otros servicios.

## ¿Cuándo y para qué se usa?

Al validar el cuerpo de una petición antes de que llegue a la lógica de negocio: que un email tenga un formato válido antes de registrar un usuario, que la cantidad de un pedido sea mayor que cero, que la fecha de fin de una reserva sea posterior a la de inicio. Cualquier dato de entrada que pueda llegar mal formado desde fuera de la aplicación es candidato a tener su propio validador.

## Lo mínimo que necesitas saber

**1. El validador es una clase separada del dato que valida**

```csharp
public class CrearPedidoCommand
{
    public int ClienteId { get; set; }
    public List<int> ProductoIds { get; set; } = new();
}

public class CrearPedidoCommandValidator : AbstractValidator<CrearPedidoCommand>
{
    public CrearPedidoCommandValidator()
    {
        RuleFor(x => x.ClienteId).GreaterThan(0);
        RuleFor(x => x.ProductoIds).NotEmpty().WithMessage("El pedido debe tener al menos un producto.");
    }
}
```

**2. Las reglas se encadenan de forma legible**

```csharp
RuleFor(x => x.Email)
    .NotEmpty()
    .EmailAddress()
    .WithMessage("El email no tiene un formato válido.");
```

**3. Reglas condicionales y dependientes de otros campos**

```csharp
RuleFor(x => x.FechaFin)
    .GreaterThan(x => x.FechaInicio)
    .WithMessage("La fecha de fin debe ser posterior a la de inicio.");

RuleFor(x => x.Descuento)
    .LessThanOrEqualTo(50)
    .When(x => x.TipoCliente == "Estandar");
```

**4. Ejecutar la validación y recoger los errores**

```csharp
var resultado = await validator.ValidateAsync(comando);
if (!resultado.IsValid)
    return BadRequest(resultado.Errors.Select(e => e.ErrorMessage));
```

**5. Registrar los validadores para que se resuelvan por inyección de dependencias**

```csharp
builder.Services.AddValidatorsFromAssemblyContaining<CrearPedidoCommandValidator>();
```

Con el validador registrado, un controlador puede pedirlo por constructor (`IValidator<CrearPedidoCommand>`) y ejecutarlo explícitamente antes de continuar, en vez de escribir a mano la validación campo a campo.

## Lo que NO hace

- **No valida reglas de negocio que dependen de estado ya guardado sin cuidado** — comprobar que un email no está registrado es posible con reglas asíncronas contra la base de datos, pero cargar el validador de demasiada lógica de negocio difumina la frontera con el caso de uso que hay detrás.
- **No sustituye al modelo de dominio** — que un pedido no pueda confirmarse sin líneas es una regla que también debe protegerse dentro de la entidad, no solo en el validador de entrada: el validador protege la frontera, el dominio protege sus propias invariantes siempre, se llame desde donde se llame.
- **No decide el código de estado HTTP** — solo produce una lista de errores; traducirla a un `400 Bad Request` u otro formato de respuesta sigue siendo responsabilidad de quien recibe la petición.

## Buenas prácticas avanzadas

- **Un validador por comando, no uno genérico reutilizado con condicionales** — `CrearPedidoCommand` y `ActualizarPedidoCommand` casi siempre tienen reglas distintas aunque compartan campos parecidos; validadores pequeños y separados son más fáciles de leer y de testear que uno grande lleno de ramas según el caso.
- **Ejecuta la validación en el borde de la aplicación, no dentro del caso de uso** — validar la forma del dato de entrada es responsabilidad de quien recibe la petición (típicamente el controlador), antes de llegar a la lógica de negocio; si el caso de uso también depende de que alguien lo haya validado antes, deja de poder invocarse con confianza desde otro sitio (un test, un job en segundo plano) sin repetir esa validación.
- **Cuidado con encadenar varias reglas asíncronas sin necesidad** — `MustAsync` para comprobar algo contra base de datos es cómodo, pero acumular varias reglas asíncronas en el mismo validador puede disparar varias consultas por cada petición; agrúpalas o reutiliza el resultado si más de una regla necesita el mismo dato.
- **No dependas del paquete de integración obsoleto para ASP.NET Core** — el paquete `FluentValidation.AspNetCore`, que antes ejecutaba la validación automáticamente sobre el *model binding*, está marcado como obsoleto por el propio proyecto; la forma recomendada hoy es resolver el validador explícitamente (por inyección de dependencias) y ejecutarlo a mano en el controlador o en un filtro propio, en vez de depender de ese "enganche" automático que ya no se mantiene.
- **Testea el validador de forma aislada, sin levantar el framework web** — al ser una clase normal (`new CrearPedidoCommandValidator()`), se puede testear pasándole un objeto de entrada y comprobando `resultado.IsValid`, sin necesidad de montar toda la petición HTTP con un `WebApplicationFactory`.

---

*En resumen: FluentValidation saca las reglas de validación del controlador y las lleva a una clase propia, legible y testeable, separada del dato que valida — sin depender de enganches automáticos ya obsoletos.*
