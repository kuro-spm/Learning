# Descripciones y ejemplos fieles

## ¿Qué es?

Las **descripciones** (`summary`, `description`) y los **ejemplos** (`example`, `examples`) de un contrato son la parte escrita para personas: explican qué hace cada operación, qué límites tiene y cómo se ven sus respuestas. Que sean **fieles** significa que dicen exactamente lo que el servidor hace, ni más ni menos.

## ¿Por qué existe?

El esquema de un contrato lo validan las máquinas; las descripciones y los ejemplos los lee gente, y nadie los valida. Por eso son lo primero que se desfasa.

Escenario: en el contrato de la tienda, un ejemplo de respuesta de error dice:

```yaml
examples:
  stockInsuficiente:
    value: { code: STOCK_INSUFICIENTE, message: No hay stock suficiente de este producto. }
```

Meses después, un cambio del servidor reformula el mensaje a «Quedan 2 unidades; has pedido 5.» y el ejemplo nadie lo toca. Quien integra la API copia el ejemplo para escribir su test, el test pasa contra un servidor simulado con ese texto, y contra el real falla. Peor aún: alguien descubre que el ejemplo ni siquiera correspondía al comportamiento original, porque se escribió de memoria.

**Una documentación que miente es peor que no tener ninguna**: quien no tiene documentación pregunta; quien tiene una equivocada se fía.

> Si ya conoces los comentarios XML de C# (`/// <summary>`), es el mismo riesgo: el comentario compila aunque diga lo contrario del código. Lo que cambia es que aquí lo leen personas de otro equipo que no pueden abrir el código para contrastar.

## ¿Cuándo y para qué se usa?

En cada operación y en cada respuesta del contrato. Importa especialmente en lo que **no se deduce del esquema**: efectos laterales, asincronía, límites, condiciones que hacen fallar una operación.

## Qué debe decir una descripción

El `summary` es una línea que nombra la operación. La `description` cuenta lo que el esquema no puede:

- **Qué hace**, con las precondiciones.
- **Qué efectos tiene** más allá de la respuesta: encola trabajo, envía un correo, cobra.
- **Si es asíncrona**: qué devuelve ahora (`202`) y dónde se consulta el resultado.
- **Qué límites tiene**: cotas, tasa de peticiones, tamaño máximo.
- **Qué errores devuelve y con qué `code`**, y en qué condiciones.

Una descripción pobre y otra útil para la misma operación:

```yaml
# Pobre
post:
  summary: Crear pedido
  description: Crea un pedido.

# Útil
post:
  summary: Crear un pedido a partir del carrito
  description: >
    Crea el pedido y **reserva el stock** de cada línea. Responde `201` con el pedido en
    estado `PENDIENTE_DE_PAGO`; el cobro es una operación aparte.

    Si alguna línea no tiene stock suficiente no se reserva **nada** (todo o nada) y responde
    `409` con `code: STOCK_INSUFICIENTE`. Máximo 50 líneas por pedido.
```

La segunda describe lo que un cliente necesita saber para usar la operación sin probar a ciegas: qué cambia en el servidor, qué pasa cuando falla y qué límite existe.

## Ejemplos que coinciden con la realidad

Un ejemplo es el contrato en acción. Tres reglas:

1. **Debe ser válido contra su propio esquema.** Un ejemplo que incumple el esquema que ilustra es un error de contrato, y se detecta con un *linter*.
2. **Debe coincidir con lo que el servidor devuelve.** Un ejemplo no es una ilustración libre: es una respuesta real simplificada.
3. **Los errores también llevan ejemplo**, nombrado por el caso (`stockInsuficiente`, `pedidoNoEncontrado`), con el `code` real.

```yaml
'409':
  description: Alguna línea no tiene stock suficiente.
  content:
    application/json:
      schema: { $ref: '#/components/schemas/Error' }
      examples:
        stockInsuficiente:
          summary: Se pide más de lo que hay
          value: { code: STOCK_INSUFICIENTE, message: Stock insuficiente. }
```

## Cómo se detecta una deriva

Hay cuatro defensas, de más barata a más cara. Conviene tener al menos las dos primeras:

| Defensa | Qué atrapa |
|---|---|
| ***Linter*** con la regla «los ejemplos validan contra su esquema» | Un ejemplo que no cumple el esquema |
| **Test de integración que valida la respuesta real contra el esquema** | Un servidor que devuelve algo distinto de lo que dice el contrato |
| **Pruebas basadas en el contrato** (por ejemplo, Schemathesis genera peticiones a partir de él) | Casos que nadie pensó en probar: valores límite, campos ausentes |
| **Revisión: un cambio de código sin cambio de contrato es sospechoso** | Un comportamiento que cambió sin que se anotara |

El segundo es muy sencillo de escribir. Con una librería que valide JSON Schema:

```csharp
[Fact]
public async Task GetProduct_RespondeSegunElContrato()
{
    var respuesta = await _client.GetAsync("/products/3f2c...");
    var json = await respuesta.Content.ReadAsStringAsync();

    var errores = _esquemaProduct.Validate(json);   // el esquema `Product` del YAML

    Assert.Empty(errores);
}
```

Si mañana alguien quita `price` de la respuesta, este test falla **antes** de que lo note nadie fuera.

## Anotar los cambios de comportamiento

Cuando el comportamiento cambia pero la forma no, la descripción es el único sitio donde se puede decir. Esa nota va con fecha y explica por qué (ver [Versionado de contratos](Versionado-de-Contratos.md)). Y si una descripción anterior queda desfasada, se corrige **y se deja constancia**, porque quien la leyó hace un mes la cree vigente.

## Buenas prácticas avanzadas

- **Referencia por `code`, no por texto.** En los ejemplos, el `code` es lo que importa y el `message` es orientativo. Pegar el mensaje literal en cinco sitios garantiza que un día estará distinto en tres.
- **No pongas números de línea del código fuente en el contrato.** «Ver `PedidoService.cs:158`» se desfasa con el siguiente commit. Nombra la clase o el método, o enlaza a un documento; el número de línea no sobrevive.
- **Documenta lo que el contrato no puede expresar y cuesta descubrir:** que una operación es asíncrona, que tiene una cuota, que un campo se rellena con retraso. Son las sorpresas que más tickets generan.
- **Explica el porqué de un límite.** «Máximo 3 variaciones por petición» invita a preguntar por qué; «máximo 3 para acotar el coste de cada petición» evita la pregunta.
- **Que cada respuesta posible tenga `description`.** Una respuesta `404` sin descripción obliga a adivinar si significa «no existe», «no tienes acceso» o ambas cosas. Y ese matiz, a veces, es una decisión de seguridad que debe estar escrita.
- **Revisa descripciones y ejemplos en cada cambio de código.** Es la pregunta que falta en la mayoría de las revisiones: «¿esto cambia algo que el contrato dice?».

## Documentación oficial

- [Learn OpenAPI: describir respuestas y ejemplos](https://learn.openapis.org/) — recorre el uso de `description`, `example` y `examples` con casos prácticos.
- [Especificación OpenAPI 3.1: Example Object](https://spec.openapis.org/oas/v3.1.0#example-object) — la fuente normativa sobre dónde puede ir un ejemplo y qué campos admite.
- [Schemathesis](https://schemathesis.readthedocs.io/) — pruebas basadas en el contrato; la guía de inicio muestra cómo apuntarlo a una API real en unos minutos.

---

*En resumen: un contrato solo es tan fiable como su parte menos vigilada; si los ejemplos y las descripciones no se comprueban contra el servidor real, acaban contando una historia que ya no es la suya.*
