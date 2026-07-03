# SDK (no es un tipo de API, pero se confunde)

## ¿Qué es?

Un **SDK** (*Software Development Kit*, kit de desarrollo de software) es un **conjunto de herramientas** —librerías, documentación, ejemplos— que un proveedor te da para usar su servicio desde un lenguaje concreto. **No es un tipo de API**: lo más habitual es que un SDK **envuelva** una API ([REST](REST.md), [gRPC](gRPC.md)...) para que sea más cómoda de usar.

## ¿Por qué existe?

Una API web es un contrato "crudo": URLs, JSON, cabeceras, errores. Usarla a mano significa construir peticiones HTTP, serializar datos, gestionar la autenticación y los reintentos tú mismo. Es repetitivo y fácil de equivocarse.

Un SDK hace ese trabajo por ti. En vez de armar una petición HTTP, llamas a un método de una clase en tu lenguaje y el SDK traduce esa llamada a la petición correcta por debajo.

> Si la API es el motor de un coche (potente pero pelado), el SDK es el coche entero con volante, pedales y salpicadero. El motor es el mismo; el SDK te da una forma cómoda y segura de conducirlo.

Por eso aparecen juntos en este tutorial: entender la diferencia evita confundir "el contrato" (API) con "la herramienta para usarlo" (SDK).

## ¿Cuándo y para qué se usa?

Cuando integras un servicio externo que ofrece SDK oficial, casi siempre conviene usarlo en lugar de llamar a la API a mano: una pasarela de pago, un servicio de almacenamiento en la nube, una plataforma de envío de emails. El SDK te ahorra detalles y reduce errores. Recurres a la API directa cuando no hay SDK para tu lenguaje o necesitas un control muy fino.

## Lo mínimo que necesitas saber

**1. Un SDK normalmente envuelve una API**

Por debajo sigue habiendo llamadas a la API; el SDK las esconde tras métodos cómodos.

```csharp
// CON SDK: una línea legible
var charge = await paymentClient.Charges.CreateAsync(amount: 5990, currency: "eur");
```

```http
# SIN SDK: la misma operación, a mano
POST https://api.pasarela.com/v1/charges
Authorization: Bearer sk_test_...
{ "amount": 5990, "currency": "eur" }
```

**2. Te da tipos y autocompletado**

Al ser código en tu lenguaje, el editor te sugiere métodos y parámetros, y los errores de tipo se ven antes de ejecutar.

```csharp
var product = await client.Products.GetAsync(id: 42);
Console.WriteLine(product.Name);   // el SDK ya sabe que Product tiene Name
```

**3. Gestiona detalles tediosos por ti**

Autenticación, reintentos, paginación, control de errores... el SDK suele encargarse de eso.

```csharp
client.Configure(apiKey: "sk_test_...", maxRetries: 3);
```

**4. Va atado a un lenguaje**

Una misma API suele tener varios SDKs oficiales: uno para C#, otro para JavaScript, otro para Python. La API es una; los SDKs, varios.

**5. No siempre existe ni hace falta**

Si no hay SDK para tu lenguaje, llamas a la API directamente. El SDK es comodidad, no obligación.

## Lo que NO hace

- **No es un tipo de API** — es una herramienta que normalmente *usa* una API; no compite con REST o GraphQL, los envuelve.
- **No añade capacidades nuevas al servicio** — solo puede hacer lo que la API por debajo permita.
- **No es universal** — cada SDK sirve para un lenguaje; cambiar de lenguaje implica otro SDK.
- **No siempre está actualizado** — a veces la API estrena funciones antes de que el SDK las incorpore.

## Buenas prácticas avanzadas

- **No dejes que los tipos del SDK invadan tu código** — si `PaymentClient` y sus clases aparecen por toda tu aplicación, cambiar de proveedor (o testear sin llamar a la API real) se vuelve una reescritura. Envuelve el SDK tras una interfaz tuya (`IPaymentGateway` con *tus* tipos) y que solo el adaptador conozca al proveedor. Es el clásico patrón *adapter*, y con SDKs de terceros se paga solo.
- **Configura timeouts y reintentos explícitamente: los defaults no son para ti** — cada SDK trae valores por defecto pensados para el caso genérico: algunos no tienen timeout, otros reintentan agresivamente operaciones que en tu dominio no son seguras de repetir (¡un cobro!). Averigua qué hace el tuyo *antes* del primer incidente y fija valores conscientes, incluyendo si los reintentos automáticos aplican a operaciones de escritura.
- **Un cliente compartido, no uno por operación** — los clientes de SDK suelen mantener por debajo un *pool* de conexiones HTTP. Crear `new PaymentClient(...)` en cada petición tira ese pool a la basura y puede agotar los sockets del sistema (el famoso *socket exhaustion* de `HttpClient` en .NET). Crea el cliente una vez (singleton o inyección de dependencias) y reutilízalo: casi todos son *thread-safe* justo para eso.
- **Fija la versión y lee el *changelog* antes de actualizar** — los SDKs cambian comportamientos sutiles entre versiones (defaults de reintento, serialización de fechas, nombres de campos) sin que tu código deje de compilar. Ancla la versión exacta en tu fichero de dependencias y trata cada actualización como un cambio funcional, no como un trámite.
- **Aprende a mirar debajo del capó** — cuando el SDK falla con un error opaco, la verdad está en la petición HTTP que construyó. Casi todos traen un modo de log/debug que muestra la petición y la respuesta crudas; actívalo y compáralas con la documentación de la API. Quien domina un SDK conoce la API que envuelve; quien solo usa el SDK se queda ciego en el primer error raro.

---

*En resumen: un SDK no es un tipo de API, sino el "coche completo" construido alrededor de una —librerías y herramientas que envuelven el contrato crudo para que lo uses cómodamente desde tu lenguaje.*
