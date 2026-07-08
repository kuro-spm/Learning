# Tipos de tests

## ¿Qué es?

Una clasificación de los tests automatizados según *cuánta* aplicación ejercitan: desde una sola clase aislada (test unitario) hasta la aplicación completa hablando con una base de datos real (test de integración o end-to-end).

## ¿Por qué existe?

No todos los errores viven en el mismo sitio. Un cálculo mal hecho se detecta probando una clase suelta; un `JOIN` mal escrito solo aparece cuando hay una base de datos de verdad; y un endpoint que devuelve 500 porque falta un servicio en la inyección de dependencias solo se ve arrancando la aplicación entera. Cada tipo de test cubre una de esas capas, y ninguno sustituye a los demás.

La imagen clásica es la **pirámide de tests**: muchos tests unitarios (rápidos y baratos) en la base, menos tests de integración en el medio, y unos pocos tests end-to-end (lentos y caros) en la cima.

> Si ya conoces los entornos de despliegue, piensa en los tipos de test como algo parecido: el unitario es tu máquina local, la integración es el entorno de staging, y el end-to-end es el ensayo general antes de producción.

## ¿Cuándo y para qué se usa?

- **Tests unitarios**: prueban una clase o método en aislamiento, sin base de datos, sin red y sin disco. Ideales para lógica de negocio pura: el cálculo del total de un pedido, la validación de un formulario de registro, una máquina de estados. Se ejecutan en milisegundos.
- **Tests de integración**: prueban que varias piezas reales funcionan juntas. Por ejemplo, que el repositorio de productos escribe y lee correctamente contra un PostgreSQL de verdad, o que las migraciones dejan el esquema como se espera.
- **Tests de API / end-to-end**: arrancan la aplicación completa (o casi) y la atacan como lo haría un cliente real: peticiones HTTP contra los endpoints, comprobando códigos de estado, JSON de respuesta y flujos completos (login → consultar perfil → logout).

En .NET, la línea entre integración y end-to-end es difusa: con `WebApplicationFactory` puedes arrancar toda la API en memoria y con Testcontainers darle una base de datos real, así que un mismo test puede cubrir ambas cosas.

## Lo mínimo que necesitas saber

**1. Un test unitario no toca infraestructura**

```csharp
[Fact]
public void CalcularTotal_ConDescuentoDel10PorCiento_AplicaElDescuento()
{
    var order = new Order(items: [new OrderItem(price: 100m, quantity: 2)]);

    var total = order.CalcularTotal(discount: 0.10m);

    Assert.Equal(180m, total);
}
```

**2. Un test de integración usa las piezas reales**

```csharp
[Fact]
public async Task Add_ConProductoNuevo_LoDejaRecuperablePorId()
{
    // _connectionString apunta a un PostgreSQL efímero (ver ficha de Testcontainers)
    var repository = new ProductRepository(_connectionString);
    var product = new Product("Teclado mecánico", price: 89.90m);

    await repository.AddAsync(product);
    var loaded = await repository.GetByIdAsync(product.Id);

    Assert.NotNull(loaded);
    Assert.Equal("Teclado mecánico", loaded.Name);
}
```

**3. Un test de API ataca los endpoints por HTTP**

```csharp
[Fact]
public async Task GET_Products_DevuelveOkConListado()
{
    var client = _factory.CreateClient(); // host en memoria, sin abrir puertos

    var response = await client.GetAsync("/api/products");

    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
}
```

**4. La estrategia de base de datos importa**

Hay dos filosofías: usar una base de datos *in-memory* (rápida pero no se comporta igual que la real) o una base de datos **real y efímera** en Docker (más lenta pero fiel). Una regla sensata: si el test valida SQL, esquema o restricciones (índices únicos, claves foráneas), usa base de datos real; si no necesita datos, arranca la aplicación sin base de datos directamente.

**5. Organiza los tests por tipo**

Una convención habitual es separar en carpetas según lo que ejercitan, con namespaces que las reflejen:

```
MyApi.Tests/
├── Api/            ← tests que atacan la app por HTTP in-memory
│   └── Helpers/    ← factories y utilidades compartidas
└── Integration/    ← tests contra base de datos real
```

## Lo que NO hace

- **No garantiza ausencia de bugs** — los tests solo demuestran que los casos que escribiste funcionan; los casos que no imaginaste siguen sin cubrir.
- **No sustituye a la revisión de código** — un test verifica comportamiento, no legibilidad ni diseño.
- **No define una frontera exacta** — la clasificación es una guía, no una taxonomía rígida; lo importante es saber qué está cubriendo (y qué no) cada test.

## Buenas prácticas avanzadas

- **Decide conscientemente qué NO mockeas** — el error clásico es mockear tanto que el test solo verifica los mocks. Si la regla de negocio vive en la base de datos (un índice único, una restricción), pruébala contra la base de datos real: mockear el repositorio ahí es probar humo.
- **Un test de arranque ("smoke test") vale oro** — un simple `GET /healthz` que devuelve 200 con la app en memoria detecta la mitad de los errores de configuración: servicios sin registrar en la inyección de dependencias, middleware mal ordenado, configuración que falta. Es el test con mejor relación coste/valor de toda la suite.
- **La pirámide es una guía, no una ley** — en una API CRUD con poca lógica de negocio, los tests de integración dan más valor que cientos de unitarios triviales. Ajusta la proporción a dónde vive tu riesgo real.
- **Ponles timeout a los tests asíncronos** — un test de integración colgado (Docker que no responde, deadlock) puede bloquear toda la suite en CI. Un `CancellationTokenSource` con timeout explícito convierte el cuelgue en un fallo diagnosticable:

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
var response = await client.GetAsync("/api/products", cts.Token);
```

## Recursos didácticos

- [The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) — el artículo de referencia sobre la pirámide, con ejemplos y matices (en inglés).
- [http.cat](https://http.cat/) — cada código de estado HTTP ilustrado con un gato; útil cuando asertas códigos de respuesta y no recuerdas qué era un 409.

---

*En resumen: los tipos de test son capas de una misma defensa — unitarios para la lógica, integración para las piezas reales, end-to-end para el conjunto; el arte está en elegir la capa adecuada para cada riesgo.*
