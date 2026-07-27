# Revisión de código generado

## ¿Qué es?

Es la revisión de código que no has escrito tú, que llega en mucha más cantidad que antes y que falla de formas distintas a las del código escrito por una persona. Sigue siendo revisión de código, pero el orden de prioridades cambia: hay errores que dejan de ser frecuentes y aparecen otros que casi no se daban.

## ¿Por qué existe?

Por dos cambios simultáneos. El primero es de **volumen**: donde antes revisabas 200 líneas al día ahora pueden llegar 2.000, y la atención no escala igual. El segundo, más sutil, es de **forma del error**: el código generado es sintácticamente impecable, con nombres coherentes, comentarios correctos y estructura limpia. Todas las señales superficiales que un revisor usa como atajo —«esto está bien escrito, seguramente esté bien pensado»— apuntan en la dirección equivocada.

Un humano cansado escribe código feo con un bug evidente. Un modelo escribe código bonito con un bug que hay que buscar. La segunda situación es más peligrosa precisamente porque no dispara ninguna alarma.

> Si has revisado alguna vez el *pull request* de una persona muy competente pero recién llegada al proyecto, es exactamente esa sensación: todo está bien escrito, y sin embargo hay tres decisiones que van contra cómo funciona el sistema porque no podía saberlas. La diferencia es que aquí eso te llega diez veces al día.

## ¿Cuándo y para qué se usa?

Siempre que vayas a integrar código generado, sin excepciones por tamaño. Y con una prioridad inversa a la intuición: **los cambios pequeños son los que peor se revisan**, porque un diff de 15 líneas invita a aprobar de un vistazo, y ahí es donde se cuelan el `double` en un importe o el `catch` vacío.

---

## Cómo falla el código generado

Esta es la taxonomía que conviene tener en la cabeza al revisar, ordenada de más a menos frecuente en la práctica.

### 1. Plausible pero inexistente

Métodos, parámetros y opciones de configuración que no existen. El modelo ha visto miles de librerías parecidas y produce la firma que *debería* existir.

```csharp
// El modelo propone esto, y parece correcto:
var productos = await _conexion.QueryAsync<Producto>(
    "SELECT * FROM Productos WHERE Estado = @estado",
    new { estado = "Activo" },
    timeout: 30);           // <-- este parámetro no existe con ese nombre

// La firma real de Dapper usa commandTimeout:
var productos = await _conexion.QueryAsync<Producto>(
    "SELECT * FROM Productos WHERE Estado = @estado",
    new { estado = "Activo" },
    commandTimeout: 30);
```

Esta clase de error es la **más fácil de detectar**: el compilador la caza. Por eso compilar es el primer filtro y no una formalidad. En lenguajes dinámicos, donde el compilador no ayuda, esta categoría se vuelve mucho más peligrosa y hay que compensarla con tests que ejecuten de verdad el camino.

### 2. Tests que no prueban nada

La categoría más traicionera, porque produce cobertura y confianza sin comprobar el comportamiento.

```csharp
// Mal: el test no puede fallar por la razón que importa
[Fact]
public async Task AplicarCupon_DevuelveCarrito()
{
    var carrito = new Carrito();
    carrito.AplicarCupon(new Cupon("VERANO10", 10));

    carrito.ShouldNotBeNull();       // siempre pasa
    carrito.Total.ShouldBe(carrito.Total);  // tautología
}

// Bien: comprueba el valor concreto que define el comportamiento
[Fact]
public async Task AplicarCupon_DescuentaDelSubtotal()
{
    var carrito = new Carrito();
    carrito.Anadir(new Producto("SKU-1", precio: 100m), cantidad: 1);

    carrito.AplicarCupon(new Cupon("VERANO10", porcentaje: 10));

    carrito.Descuento.ShouldBe(10m);
    carrito.Total.ShouldBe(90m);
}
```

El test malo pasa el 100 % de las veces, incluso si `AplicarCupon` está vacío. **La comprobación al revisar un test es siempre la misma: ¿por qué razón concreta fallaría este test?** Si no sabes contestarla, el test no sirve. Un truco rápido y muy eficaz: rompe la implementación a propósito —cambia el signo, devuelve cero— y comprueba que el test se pone rojo. Si sigue verde, es decorativo.

### 3. Sobreingeniería y código no pedido

Pediste una validación y llegan una interfaz, una fábrica, un registro en el contenedor de dependencias y tres opciones de configuración.

```csharp
// Pedido: "valida que la cantidad sea mayor que cero"
// Llega:
public interface ICantidadValidator { ValidationResult Validar(int cantidad); }

public class CantidadValidatorFactory
{
    public ICantidadValidator Crear(ValidationStrategy estrategia) => estrategia switch
    {
        ValidationStrategy.Strict => new StrictCantidadValidator(),
        ValidationStrategy.Lenient => new LenientCantidadValidator(),
        _ => throw new ArgumentOutOfRangeException(nameof(estrategia)),
    };
}

// Lo que hacía falta:
if (cantidad <= 0)
    return Resultado.Error("La cantidad debe ser mayor que cero.");
```

No es un error funcional —seguramente funciona— y por eso pasa las revisiones. El coste llega después: cuatro tipos que mantener, un punto de extensión que nadie va a usar y un `ValidationStrategy.Lenient` que nadie sabe qué significa. Al revisar, la pregunta es: **¿cada abstracción aquí tiene hoy al menos dos usos reales?** Si tiene uno, es especulación.

### 4. Manejo de errores fantasma

Código defensivo contra situaciones imposibles, y captura de excepciones que oculta problemas reales.

```csharp
// Mal
public async Task<Producto?> ObtenerAsync(int id)
{
    try
    {
        return await _repositorio.ObtenerPorIdAsync(id);
    }
    catch (Exception)
    {
        return null;    // un timeout de base de datos ahora es "no existe"
    }
}
```

Un fallo de conexión y un producto inexistente pasan a ser indistinguibles, y el bug aparecerá tres capas más arriba como un 404 inexplicable. La revisión aquí busca dos patrones: `catch` que capturan `Exception` a secas, y validaciones de cosas que el sistema de tipos o el framework ya garantizan.

### 5. Duplicación en lugar de reutilización

El modelo no siempre sabe que ya existe un helper que hace eso. Si su contexto no incluía `Utilidades/FormatoImportes.cs`, escribirá su propia versión, ligeramente distinta.

Es la categoría que las herramientas automáticas peor detectan y en la que un revisor con conocimiento del proyecto es irremplazable. Es también la razón por la que dar contexto al agente —los ficheros relevantes, las convenciones— no es opcional: la duplicación se previene antes, no se corrige después.

### 6. Rendimiento: el N+1 y compañía

```csharp
// Mal: una consulta por pedido
var pedidos = await _conexion.QueryAsync<Pedido>("SELECT * FROM Pedidos WHERE ClienteId = @id", new { id });
foreach (var pedido in pedidos)
    pedido.Lineas = (await _conexion.QueryAsync<LineaPedido>(
        "SELECT * FROM LineasPedido WHERE PedidoId = @pedidoId", new { pedidoId = pedido.Id })).ToList();

// Bien: dos consultas, independientemente del número de pedidos
var pedidos = (await _conexion.QueryAsync<Pedido>(
    "SELECT * FROM Pedidos WHERE ClienteId = @id", new { id })).ToList();
var lineas = await _conexion.QueryAsync<LineaPedido>(
    "SELECT * FROM LineasPedido WHERE PedidoId IN @ids", new { ids = pedidos.Select(p => p.Id) });
```

La versión mala es correcta funcionalmente y pasa todos los tests, porque en los tests hay tres pedidos. Con 500, tumba la página. Los tests no protegen de esto: hay que verlo leyendo, o medirlo.

### 7. Seguridad

Las de siempre, con dos particularidades: aparecen con más frecuencia en el código generado y llegan con mejor aspecto.

```csharp
// SQL construido por concatenación
var sql = $"SELECT * FROM Productos WHERE Nombre LIKE '%{busqueda}%'";  // inyección

// Secretos en el código
private const string ApiKey = "sk-ant-api03-...";  // acaba en Git para siempre

// Endpoint nuevo sin autorización, porque no lo pediste explícitamente
[HttpDelete("/api/productos/{id}")]
public async Task<IActionResult> Eliminar(int id) { ... }   // falta [Authorize]
```

El último es el más típico: **el modelo implementa lo que pediste, y la autorización no estaba en lo que pediste.** No es un fallo del modelo, es un hueco de la especificación. Merece una comprobación explícita en la revisión de cualquier endpoint nuevo. Para el manejo correcto de credenciales, la referencia está en [Gestión de secretos en desarrollo](../../seguridad/gestion-de-secretos-en-desarrollo/README.md).

---

## El sesgo de plausibilidad

Merece su propio apartado porque es el problema central de esta ficha.

Todos los revisores usan atajos: si el código está bien formateado, tiene buenos nombres y comentarios coherentes, se lee con menos desconfianza. Es un heurístico razonable con código humano —quien escribe con cuidado la forma suele haber pensado el fondo— y es **completamente inválido** con código generado, donde la forma es siempre buena por construcción y no dice nada del fondo.

Tres contramedidas prácticas:

1. **Lee el diff, nunca el resumen.** El resumen que produce el agente describe lo que *pretendía* hacer. Los bugs están, por definición, en la diferencia entre la intención y el código.
2. **Revisa el código de test antes que el de implementación.** Si los tests son buenos, gran parte de la implementación queda garantizada. Si son decorativos, ya sabes que no puedes fiarte de que los tests estén en verde.
3. **Pon un límite al tamaño de lo que aceptas de una vez.** Si no puedes revisar 600 líneas con atención real, no las aceptes de golpe: pide la implementación por pasos con *commit* por paso (ver [Desarrollo dirigido por especificación](Desarrollo-Dirigido-por-Especificacion.md)). Aprobar sin revisar es peor que no revisar, porque queda registrado como revisado.

---

## Primero las máquinas, después las personas

Todo lo que una herramienta determinista pueda comprobar, no debe consumir atención humana. El orden importa, de más barato a más caro:

```bash
dotnet build                      # 1. ¿existe lo que invoca?
dotnet format --verify-no-changes # 2. ¿respeta el estilo del proyecto?
dotnet test                       # 3. ¿rompe algo que estaba cubierto?
dotnet test /p:CollectCoverage=true  # 4. ¿lo nuevo está cubierto?
```

Ejecutar esto **antes** de leer el diff cambia lo que buscas al leer: sabes que compila y que los tests pasan, así que puedes concentrarte en lo que ninguna herramienta ve —si el diseño es el correcto, si falta autorización, si hay un N+1, si esa abstracción se justifica.

Y lo mejor es que el agente ejecute estos comandos por sí mismo antes de darte nada (ver [Agentes de codificación](Agentes-de-Codificacion.md)): cuando el bucle está cerrado, la categoría 1 entera desaparece de tu mesa.

---

## Usar un LLM para revisar código de un LLM

Funciona, con matices importantes. Un revisor automático encuentra cosas reales, sobre todo en cambios grandes donde la atención humana se agota. Pero hay que pedírselo de una forma concreta.

El error más común es **filtrar por severidad en el mismo paso en que se buscan los problemas**:

```python
# Mal: el filtro suprime hallazgos reales
PROMPT_MAL = """Revisa este diff. Reporta solo los problemas importantes
y de alta severidad. No seas quisquilloso."""

# Bien: cobertura primero, filtro después
PROMPT_BIEN = """Revisa este diff y reporta TODOS los problemas que
encuentres, incluidos los que te parezcan de baja severidad o de los que
no estés seguro. No filtres por importancia en este paso.

Para cada hallazgo indica:
- fichero y línea
- qué falla, en una frase
- un escenario concreto en el que se manifieste (entradas -> resultado incorrecto)
- tu nivel de confianza (alto / medio / bajo)

Si no encuentras nada, responde exactamente "Sin hallazgos"."""
```

Los modelos actuales siguen las instrucciones muy literalmente: si les dices «solo lo importante», investigan igual de bien pero **se callan** lo que juzgan por debajo del umbral, y tú pierdes hallazgos válidos sin saberlo. Pedir cobertura con confianza declarada y filtrar tú después da muchos más problemas reales encontrados.

El otro elemento clave es exigir un **escenario concreto** por hallazgo. Es el mejor filtro contra falsos positivos: un problema que no se puede acompañar de un caso donde falle suele ser una preocupación teórica.

Sus límites, que conviene tener claros:

- **No conoce tu contexto de negocio.** No puede saber que ese endpoint debe ser accesible sin autenticación por un requisito legal.
- **No sabe qué existe ya en el proyecto** salvo que le pases esos ficheros. Es malo detectando duplicación.
- **Comparte los sesgos del generador.** Un revisor y un generador de la misma familia pueden considerar razonable la misma abstracción innecesaria. Por eso el revisor automático complementa la revisión humana; no la sustituye.

---

## Una lista de comprobación que funciona

Ordenada por relación entre hallazgos y esfuerzo:

- [ ] **Compila, pasa el *linter* y pasa los tests.** Automatizado, antes de leer nada.
- [ ] **¿Por qué razón concreta fallaría cada test nuevo?** Si no hay respuesta, el test es decorativo.
- [ ] **¿Los tipos son los correctos?** `decimal` para dinero, no `double`; fechas con zona horaria cuando importa.
- [ ] **¿Hay `catch` que capturen `Exception` a secas o que se traguen el error?**
- [ ] **¿Cada abstracción nueva tiene al menos dos usos reales hoy?**
- [ ] **¿Hay consultas dentro de bucles?**
- [ ] **¿Los endpoints nuevos tienen autorización?**
- [ ] **¿Hay algún secreto, URL interna o credencial en el diff?**
- [ ] **¿Reimplementa algo que ya existe en el proyecto?**
- [ ] **¿Está en el alcance que se pidió, sin extras?**

Y la que cierra todas: **¿podrías defender cada línea de este diff en una revisión, explicando por qué está así?** Si la respuesta es «lo escribió el agente», el cambio no está revisado. Quien lo integra lo firma; la autoría no transfiere la responsabilidad.

---

## Buenas prácticas avanzadas

- **Revisa los tests antes que la implementación, y verifica que pueden fallar.** Es la inversión de esfuerzo más rentable de toda la revisión: rompe la implementación a mano y comprueba que el test se pone rojo. Un test que sigue verde con la implementación rota es peor que no tener test, porque además genera confianza falsa y cobertura contabilizada.
- **Desconfía especialmente de los diffs pequeños y bonitos.** La atención se relaja con 15 líneas limpias, y ahí es donde viven el `double` en un importe, el `catch` vacío y el endpoint sin `[Authorize]`. Los diffs grandes al menos provocan cautela; los pequeños se aprueban de un vistazo. Aplica la lista completa igual, sobre todo cuando el cambio parece trivial.
- **Al pedir revisión automática, exige cobertura y confianza, nunca un filtro de severidad.** «Reporta solo lo importante» hace que el modelo encuentre lo mismo y te cuente menos —te bajas la detección sin bajarte los bugs. Pide todo con nivel de confianza y escenario de fallo concreto, y filtra tú en un segundo paso; el escenario concreto es además el mejor descarte de falsos positivos.
- **Cuenta las abstracciones nuevas y aplica la regla de los dos usos.** Interfaces con una sola implementación, fábricas que construyen un único tipo, opciones de configuración con un valor posible: cada una es deuda inmediata a cambio de flexibilidad que quizá nunca haga falta. Al revisar código generado esta categoría es la más frecuente de las que *no* rompen nada, y la que más pesa a seis meses.
- **Comprueba de forma explícita lo que no pediste y aun así hace falta.** Autorización, validación de entrada, límites de tamaño, tiempos de espera, idempotencia. El modelo implementa el alcance que le diste; los requisitos transversales solo aparecen si estaban escritos. Convierte esa lista en una comprobación fija de tu revisión de cualquier endpoint o *handler* nuevo.
- **Cierra el bucle en el agente para que las categorías mecánicas no lleguen a tu mesa.** Si el agente compila, formatea y ejecuta tests antes de entregar, los errores de API inventada y de estilo desaparecen del diff que revisas, y tu atención se va entera a lo que ninguna herramienta puede ver. Un bucle abierto convierte a un revisor humano en un compilador lento.

## Recursos didácticos

- **[Google Engineering Practices — Code Review](https://google.github.io/eng-practices/review/)** — la guía de revisión más usada de la industria. Anterior a los LLM y perfectamente aplicable: buena parte del valor está en la jerarquía de qué mirar primero.
- **[OWASP Top Ten](https://owasp.org/www-project-top-ten/)** — el catálogo de referencia de vulnerabilidades web. Útil como lista de comprobación de seguridad en cualquier endpoint nuevo, generado o no.
- **[Mutation testing con Stryker.NET](https://stryker-mutator.io/docs/stryker-net/introduction/)** — automatiza la pregunta «¿pueden fallar mis tests?»: rompe el código a propósito y te dice qué mutaciones no detecta ningún test. Es la respuesta industrial al problema de los tests decorativos.

---

*En resumen: el código generado no falla por ser feo, falla por ser plausible — así que revisa los tests antes que el código, exige que puedan fallar, y no confíes en ninguna señal superficial de calidad.*
