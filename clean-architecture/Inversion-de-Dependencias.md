# Inversión de Dependencias

## ¿Qué es?

La inversión de dependencias es el mecanismo que permite que una capa interna "use" algo de una capa externa **sin depender de ella**. En lugar de que el código de negocio dependa de la base de datos, ambos dependen de una **interfaz** (un contrato) definida por el código de negocio.

## ¿Por qué existe?

Imagina que tu lógica de negocio necesita guardar un pedido. Lo natural sería que llamara directamente a la base de datos. Pero entonces la lógica quedaría **atada** a esa base de datos: cambiarla obligaría a tocar el negocio, y probarlo exigiría una base de datos real.

La inversión de dependencias da la vuelta a esa relación. El negocio no dice "usa SQL Server"; dice "necesito *algo* que sepa guardar pedidos", y describe ese algo con una interfaz. Quién la cumple (SQL Server, un fichero, un objeto falso de test) se decide fuera. Así la flecha de dependencia, que apuntaría hacia fuera, queda **invertida** y apunta hacia dentro.

> Si ya conoces el concepto de "enchufe y toma de corriente": el negocio define la forma del enchufe (la interfaz); cualquier aparato que encaje (la implementación) sirve, y el negocio no necesita saber cuál es.

## ¿Cuándo y para qué se usa?

Aparece cada vez que el centro de la aplicación necesita un servicio que vive en el borde: guardar y leer datos, enviar un email, cobrar con una pasarela de pago, consultar una API externa. En todos esos casos defines una interfaz en el centro y la implementas en la infraestructura.

Es también la base de la **inyección de dependencias**: como el caso de uso pide una interfaz, alguien (normalmente un contenedor de servicios) le entrega la implementación concreta al arrancar la aplicación.

## Lo mínimo que necesitas saber

**1. El centro define el contrato (la interfaz)**

```csharp
// Vive en la capa de negocio. No menciona ninguna tecnología.
public interface IPasarelaPago
{
    Task<bool> Cobrar(decimal importe, string tarjeta);
}
```

**2. El borde implementa el contrato**

```csharp
// Vive en la infraestructura, con la tecnología real (Stripe, PayPal...).
public class PasarelaPagoStripe : IPasarelaPago
{
    public Task<bool> Cobrar(decimal importe, string tarjeta) { /* llamada a Stripe */ }
}
```

**3. El caso de uso depende de la interfaz, no de la implementación**

```csharp
public class ProcesarCompra(IPasarelaPago pasarela)
{
    public Task<bool> Ejecutar(decimal total, string tarjeta)
        => pasarela.Cobrar(total, tarjeta);
}
```

`ProcesarCompra` no sabe si por detrás hay Stripe, PayPal o un simulador. Solo conoce el contrato.

**4. Alguien conecta las piezas al arrancar**

```csharp
// En la configuración de la aplicación (la capa más externa)
services.AddScoped<IPasarelaPago, PasarelaPagoStripe>();
```

Cambiar de Stripe a PayPal es cambiar **esta línea**, sin tocar la lógica de negocio. Y en los tests pasas una implementación falsa que devuelve `true` sin cobrar nada real.

## Lo que NO hace

- **No es lo mismo que la inyección de dependencias** — la inversión es el *principio* (depender de abstracciones); la inyección es una *técnica* habitual para aplicarlo.
- **No requiere un contenedor de dependencias** — puedes pasar las implementaciones a mano; el contenedor solo lo automatiza.
- **No significa "crear una interfaz para todo"** — solo tiene sentido en las fronteras entre capas, no en cada clase interna.

## Buenas prácticas avanzadas

- **Diseña la interfaz desde la necesidad del negocio, no calcando al proveedor** — si tu `IPasarelaPago` tiene un método `CreatePaymentIntent` y devuelve los objetos del SDK de Stripe, no has invertido nada: le has puesto a Stripe un disfraz. El contrato habla el idioma del centro (`Cobrar(importe, tarjeta)`); traducir eso a lo que pida el proveedor es trabajo de la implementación.
- **Detecta las abstracciones con fugas por sus firmas** — una interfaz que expone `DbConnection`, `HttpResponseMessage` o excepciones del proveedor arrastra la tecnología hacia el centro por la puerta de atrás. Prueba del algodón: ¿podrías implementar el contrato con un diccionario en memoria sin cambiar su firma? Si no, tiene fugas.
- **Interfaces pequeñas por rol, no espejos de la clase** — la interfaz de quince métodos que replica 1:1 su única implementación no abstrae, solo duplica. Corta por lo que cada consumidor necesita (quien solo lee pedidos no necesita ver `Guardar`): las interfaces pequeñas hacen los fakes de test triviales y los cambios, locales.
- **Cuidado con los lifetimes del contenedor** — inyectar un servicio `Scoped` (por ejemplo, algo que arrastra el contexto de base de datos de la petición) dentro de un `Singleton` deja esa instancia "de petición" atrapada para toda la vida de la aplicación (*captive dependency*): funciona en la demo y en producción mezcla datos entre usuarios. Revisa los lifetimes al registrar, no cuando explote.

---

*En resumen: la inversión de dependencias hace que el negocio dependa de un contrato que él mismo define, no de la tecnología que lo cumple — así la flecha de dependencia apunta hacia dentro y el centro queda libre de detalles técnicos.*
