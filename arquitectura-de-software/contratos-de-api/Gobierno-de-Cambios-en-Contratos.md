# Gobierno de cambios en contratos

## ¿Qué es?

El **gobierno de cambios** es el conjunto de reglas sobre **cómo y quién** puede modificar un contrato de API: qué se revisa, quién da el visto bueno, qué se comprueba de forma automática y qué ha de ocurrir en el mismo cambio (subir la versión, regenerar clientes, actualizar tests). No es burocracia por sí misma: es lo que hace que un contrato siga siendo un acuerdo y no una opinión del último que lo tocó.

## ¿Por qué existe?

Un contrato cambia por dos caminos: **a propósito**, cuando alguien decide que la API debe ser distinta, y **por accidente**, cuando un cambio de código altera la API sin que nadie se dé cuenta. El gobierno de cambios existe para que el segundo camino se parezca lo máximo posible al primero.

Sin él, ocurre lo habitual: alguien modifica el YAML en una rama, otra persona cambia el servidor para que haga lo que cree que dice, el cliente se regenera al día siguiente, y cuando quien consume lo nota la API ya es otra.

> Si ya conoces las *pull requests* con revisión obligatoria y las ramas protegidas, es la misma idea aplicada a un tipo de fichero especial: uno que, al cambiar, afecta a gente que no está en el repositorio.

## ¿Cuándo y para qué se usa?

Cuanto más lejos esté quien consume de quien implementa, más necesario. Un contrato interno con un solo cliente en el mismo repositorio puede gobernarse con una revisión normal; uno que usan otros equipos o clientes externos necesita puerta de aprobación explícita.

## El flujo de un cambio

```
 propuesta        revisión        aprobación       implementación     verificación
 (YAML en una  →  de quien     →  de quien       →  servidor y     →  tests y CI
  rama)           consume e       decide            cliente          comprueban
                  implementa                        regenerado       el contrato
```

Paso a paso, con la tienda online como ejemplo (se añade el campo `brand` a `Product`):

1. **Propuesta.** Una rama con el cambio **solo en el contrato**: el campo, su descripción, la versión subida de `1.2.0` a `1.3.0` y la nota de versión.
2. **Revisión.** Quien consume y quien implementa leen el YAML. Es barato corregir aquí; cuesta diez veces más después de escribir el servidor.
3. **Aprobación.** Alguien con autoridad sobre la API dice «sí» de forma explícita.
4. **Implementación.** El servidor y el cliente generado se actualizan **a partir del contrato aprobado**, no al revés.
5. **Verificación.** Los tests y la integración continua comprueban que lo implementado cumple lo aprobado.

## Qué se aprueba y qué pasa si cambia después

Lo que se aprueba es **un texto concreto**. Si después de la aprobación cambia el diseño, el texto aprobado deja de ser el vigente y hay que volver a aprobar. Es un caso muy común y fácil de pasar por alto:

- El miércoles se aprueba «todas las variantes del producto parten de la imagen base».
- El jueves, al probarlo, se decide que solo lo hagan las variantes de tipo A.
- El contrato publicado dice lo del miércoles y el servidor hace lo del jueves.

La salida honesta es corregir el texto, volver a pedir el visto bueno **sobre el texto nuevo**, y dejar constancia de que hubo una primera redacción descartada (ver [Versionado de contratos](Versionado-de-Contratos.md)). Dejar «aprobado» sobre un texto que ya no describe lo que se hace es peor que no haber pedido la aprobación.

Conviene que **la aprobación quede escrita** en el propio contrato, con persona y fecha:

```yaml
description: >
  **v1.3.0 (2026-03-12)** — campo opcional `brand` en `Product`. Aprobado por
  <persona responsable de la API> el 2026-03-12.
```

## Qué se comprueba de forma automática

Lo que una persona tiene que recordar, tarde o temprano se olvida. Lo que ejecuta la integración continua, no. Cuatro comprobaciones que merecen la pena:

| Comprobación | Herramienta de ejemplo | Qué atrapa |
|---|---|---|
| El contrato sigue las reglas de estilo | Spectral | Operaciones sin `operationId`, ejemplos que no validan |
| El cambio no rompe a nadie (comparado con la versión anterior) | `oasdiff` | Un campo eliminado, un tipo cambiado, una validación más estricta |
| El cliente generado está al día | Regenerar y `git diff --exit-code` | Contrato modificado sin regenerar los tipos |
| La API real cumple el contrato | Tests de integración que validan las respuestas | Código que se desvía de lo aprobado |

Un ejemplo de las dos primeras en un *workflow* de integración continua:

```yaml
- name: Lint del contrato
  run: npx @stoplight/spectral-cli lint contratos/catalogo.openapi.yaml

- name: ¿Rompe respecto a la rama principal?
  run: oasdiff breaking origin/main:contratos/catalogo.openapi.yaml contratos/catalogo.openapi.yaml --fail-on ERR
```

Y la tercera, para que el cliente no se quede atrás:

```yaml
- name: Cliente generado al día
  run: |
    npx openapi-typescript contratos/catalogo.openapi.yaml -o src/api/catalogo.ts
    git diff --exit-code src/api/catalogo.ts
```

Una ruptura detectada no tiene por qué impedir el cambio, pero **sí obliga a una decisión consciente**: es el momento de subir la versión mayor o de replantear el cambio.

## Tests que fijan el contrato

Un test de contrato comprueba **la API contra el contrato**, no contra la implementación. Dos tipos útiles:

- **De forma:** la respuesta real valida contra el esquema (ver [Descripciones y ejemplos fieles](Descripciones-y-Ejemplos-Fieles.md)).
- **De garantías:** lo que el contrato promete y el esquema no expresa. Por ejemplo, que un error de validación no devuelve el valor que falló, o que una operación sin permiso responde `403` sin revelar si el recurso existe.

```csharp
[Fact]
public async Task PostPedido_SinPermiso_Devuelve403_ConCodeEstable()
{
    var respuesta = await _clienteSinPermiso.PostAsJsonAsync("/orders", PedidoValido());

    Assert.Equal(HttpStatusCode.Forbidden, respuesta.StatusCode);
    var error = await respuesta.Content.ReadFromJsonAsync<ErrorResponse>();
    Assert.Equal("FORBIDDEN", error!.Code);
}
```

## Qué hacer cuando el contrato y el código discrepan

Pasará. La regla es una sola: **el contrato aprobado manda**. Si el código difiere, o se corrige el código o se cambia el contrato **por el mismo proceso**; nunca se deja la diferencia sin decidir. Lo peor que puede ocurrir es descubrir la discrepancia, no saber cuál de los dos tiene razón y dejarla como está «de momento».

## Condicionar la publicación a una comprobación

A veces el contrato está aprobado pero una **comprobación** decide si el cambio se publica: una prueba de rendimiento, una medida de calidad, una verificación contra un servicio externo. Es un patrón legítimo, con dos condiciones:

- Se escribe **antes** de implementar qué resultado permite publicar y cuál lo impide.
- Si la comprobación no se puede hacer como se pensó, se anota tal cual —«se validó a ojo, no se calculó la cifra»— y no se inventa una medida. Un criterio incumplido y reconocido se puede discutir; uno maquillado no.

## Buenas prácticas avanzadas

- **El contrato se aprueba antes del código, no después.** Aprobar un contrato cuando el servidor ya está escrito convierte la revisión en un trámite: ya nadie va a pedir un cambio que obligue a rehacer trabajo.
- **Un cambio de contrato por *pull request*, cuando sea posible.** Si un mismo cambio añade un campo, renombra otro y corrige una descripción, quien revisa no puede aprobar una parte y rechazar otra.
- **Regenera siempre desde el contrato del mismo *commit*.** Un cliente generado desde una rama distinta a la del servidor que va a usar es una fuente de errores difíciles de reproducir.
- **Registra quién aprobó y cuándo en el contrato mismo**, no en un chat. Dentro de seis meses nadie encontrará la conversación; el YAML siempre estará ahí.
- **Una ruptura es una conversación, no un número.** Subir a `2.0.0` no sustituye avisar a quien consume, darle plazo y mantener la versión anterior mientras migra.
- **Los *tests* de contrato se ejecutan contra el contrato, no contra una copia en el código de prueba.** Si el test lleva su propia copia del esquema, valida contra lo que alguien escribió una vez.

## Documentación oficial

- [oasdiff](https://github.com/oasdiff/oasdiff) — la herramienta de la sección de comprobaciones; el README lista qué cambios clasifica como ruptura.
- [Spectral](https://github.com/stoplightio/spectral) — *linter* de contratos; la sección de *rulesets* explica cómo codificar las reglas propias del equipo.
- [Cómo versiona Stripe su API](https://stripe.com/blog/api-versioning) — un ejemplo de gobierno de cambios llevado muy lejos: cada cambio con su versión, su compatibilidad y su documentación.
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines) — incluye un proceso de revisión de cambios de API y una definición de qué se considera ruptura.

---

*En resumen: un contrato fiable es el que cambia a propósito —se propone, se revisa, se aprueba y se verifica— y no el que se desliza con cada cambio de código.*
