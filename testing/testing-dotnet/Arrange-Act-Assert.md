# Arrange-Act-Assert

**¿Qué es?** Un patrón de estructura para escribir tests en tres bloques ordenados: **Arrange** (preparar el escenario), **Act** (ejecutar la acción que se prueba) y **Assert** (comprobar el resultado). Es la gramática universal de los tests: casi cualquier test bien escrito, en cualquier lenguaje, sigue esta forma.

---

## ¿Por qué existe?

Un test sin estructura mezcla preparación, ejecución y comprobaciones, y al fallar no se sabe qué parte del escenario era relevante. AAA obliga a responder tres preguntas en orden: ¿de qué situación parto?, ¿qué acción pruebo?, ¿qué esperaba que pasara? Es la misma idea que el *Given-When-Then* de BDD, con otros nombres.

---

## ¿Cuándo y para qué se usa?

En todos los tests, siempre. La convención habitual en C# es marcar los bloques con comentarios literales, que actúan como separadores visuales:

```csharp
[Fact]
public async Task GET_Health_DevuelveOk()
{
    // Arrange: cliente HTTP contra el host en memoria
    var client = _factory.CreateClient();
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

    // Act
    var response = await client.GetAsync("/healthz", cts.Token);

    // Assert
    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
}
```

Cuando la acción y la comprobación son inseparables (probar que algo lanza excepción), se admite el bloque combinado `// Act + Assert`:

```csharp
// Act + Assert: insertar un email duplicado viola el índice único
var ex = Assert.Throws<PostgresException>(() => connection.Execute(insertDuplicado));
Assert.Equal("23505", ex.SqlState); // unique_violation
```

---

## Lo mínimo que necesitas saber

**1. Un solo Act por test** — si tu test tiene dos bloques Act, son dos tests. La excepción legítima son los tests de *flujo* end-to-end (login → perfil → logout), donde el flujo completo ES el escenario que se prueba.

**2. El Arrange puede vivir fuera del test** — constructores de la clase de test y fixtures compartidas son "Arrange global". El comentario `// Arrange` del test marca solo la preparación específica de ese caso.

**3. El Assert debe ser concreto** — comprueba el valor esperado, no solo que "no explotó". `Assert.NotNull(result)` a secas es un test que aprueba casi cualquier bug.

---

## Lo que NO hace

- **No dicta cuántas aserciones puede haber** — varias `Assert` seguidas sobre el mismo resultado son un solo bloque Assert perfectamente válido.
- **No sustituye a un buen nombre de test** — el patrón ordena el cuerpo; el nombre (`Metodo_Escenario_Resultado`) cuenta la historia desde fuera.

---

## Buenas prácticas avanzadas

- **Si el Arrange ocupa 20 líneas, el diseño te está hablando** — o la clase bajo test tiene demasiadas dependencias, o te falta un helper/builder que construya el escenario en una línea. Los mejores equipos tratan el Arrange largo como un *code smell* del código de producción, no del test.
- **Los comentarios `// Arrange` explican el *porqué*, no el *qué*** — `// Arrange: cliente contra el host en memoria (no sale a la red)` aporta contexto; `// Arrange: creamos el cliente` es ruido. Úsalos para dejar constancia de decisiones (por qué este timeout, por qué esta configuración).
- **Assert sobre el contrato, no sobre la implementación** — comprueba lo que un consumidor observaría (el JSON devuelto, el código HTTP, la fila en la base de datos), no detalles internos que puedan cambiar sin romper nada. Un test acoplado a la implementación falla con cada refactor y acaba ignorado.

---

*En resumen: Arrange-Act-Assert es la disciplina de que cada test cuente una historia completa en tres actos — sitúa, actúa y comprueba.*
