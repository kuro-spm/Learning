# Domain-Driven Design

> 🧭 **Tutorial recomendado de forma proactiva:** este contenido no nace de una necesidad concreta detectada en un proyecto, sino de una sugerencia para ampliar la colección.

## ¿Qué es?

Domain-Driven Design (DDD) es un enfoque para diseñar software complejo poniendo el **modelo del negocio** en el centro de las decisiones técnicas, y construyendo ese modelo junto a las personas expertas en el negocio, usando su mismo vocabulario.

## ¿Por qué existe?

En sistemas grandes es habitual que una misma palabra signifique cosas distintas según quién la use: "cliente" para el equipo de ventas no es lo mismo que "cliente" para el equipo de soporte o el de facturación. Si el código intenta modelar un único "Cliente" universal que sirva para todos, termina lleno de campos opcionales y casos especiales que no encajan con nadie.

DDD parte de aceptar esa realidad en lugar de luchar contra ella: en vez de un modelo único para toda la aplicación, propone dividirla en **contextos** más pequeños, cada uno con su propio modelo, consistente puertas adentro, y con un vocabulario compartido entre quienes programan y quienes conocen el negocio.

> Piensa en los departamentos de una empresa: "pedido" para el almacén (bultos que preparar) no es lo mismo que "pedido" para contabilidad (una factura pendiente de cobro). Cada departamento tiene razón dentro de su propio contexto; el problema aparece solo si alguien intenta forzar una única definición para todos.

## ¿Cuándo y para qué se usa?

DDD tiene sentido cuando el negocio es **complejo**: muchas reglas, muchas excepciones, y varios subsistemas que hablan de "lo mismo" con matices distintos. Una tienda online mediana o grande es un buen ejemplo: el catálogo, los pedidos, los envíos y la facturación pueden convivir como partes separadas de la misma aplicación, cada una con su propio modelo de lo que es un "producto" o un "cliente".

Para un CRUD sencillo (una lista de tareas, un formulario de contacto) DDD es sobrecoste: no hay suficiente complejidad de negocio que justifique dividir el dominio en contextos ni invertir tiempo en acordar un vocabulario compartido.

## Lo mínimo que necesitas saber

**1. Lenguaje ubicuo**

Es el vocabulario que comparten el código y las personas expertas en el negocio, dentro de un mismo contexto. Si en las reuniones se habla de "confirmar un pedido", el código no debería llamar a ese método `SetStatus(2)`, sino `pedido.Confirmar()`. El nombre de las clases, métodos y variables debe poder leerse en voz alta delante de alguien de negocio sin traducir nada.

```csharp
// El lenguaje ubicuo se refleja directamente en el código
public class Pedido
{
    public void Confirmar() { /* ... */ }
    public void Cancelar(string motivo) { /* ... */ }
}
```

**2. Bounded context (contexto delimitado)**

Es la frontera explícita dentro de la cual un modelo es válido y consistente. Dentro del contexto de Catálogo, un "Producto" tiene precio, descripción e imágenes. Dentro del contexto de Envíos, ese mismo "Producto" es solo un peso y unas dimensiones para calcular un paquete. Son dos modelos distintos del mismo concepto del mundo real, y eso está bien: cada uno resuelve un problema diferente.

```text
Contexto Catálogo   → Producto { Precio, Descripcion, Imagenes }
Contexto Envíos      → Producto { PesoKg, Alto, Ancho, Largo }
```

**3. Las piezas tácticas: entidades, objetos de valor, agregados, repositorios y eventos**

Dentro de cada contexto, DDD propone un conjunto de "bloques de construcción" para modelar el dominio: entidades con identidad, objetos de valor inmutables, agregados que agrupan varias entidades bajo una misma regla de consistencia, repositorios para cargarlos y guardarlos, y eventos de dominio para comunicar que algo ha pasado. Cada uno tiene su propia ficha en esta colección: [Value Objects](Value-Objects.md), [Entidades y Agregados](Entidades-y-Agregados.md), [Repositorios](Repositorios.md) y [Domain Events](Domain-Events.md).

**4. Relación con Clean Architecture**

DDD y Clean Architecture responden a preguntas distintas y se complementan de maravilla: DDD te dice **qué** poner en el centro de tu aplicación (un modelo de dominio rico, con lenguaje del negocio); Clean Architecture te dice **dónde** ponerlo y cómo protegerlo del exterior (la [regla de dependencia](../clean-architecture/Regla-de-Dependencia.md), las capas). En la práctica, el modelo de dominio de DDD es justo lo que vive en la [capa de dominio](../clean-architecture/Capa-de-Dominio.md) de Clean Architecture. Si no conoces esa colección, es buen momento para leer [Clean Architecture](../clean-architecture/README.md) antes de seguir con las siguientes fichas de patrones.

## Lo que NO hace

- **No es una tecnología ni un framework** — es una forma de pensar el diseño; no hay nada que instalar.
- **No obliga a aplicar todo el dominio con las mismas reglas** — puedes tener contextos "ricos" en DDD y otros contextos simples de puro CRUD conviviendo en la misma aplicación.
- **No dice cómo deben comunicarse los contextos entre sí** — eso es una decisión aparte (llamadas directas, eventos, una API), aunque suele resolverse con los mismos [Domain Events](Domain-Events.md).
- **No sustituye a Clean Architecture** — la complementa: una da la estructura en capas, la otra da el contenido de la capa de dominio.

## Buenas prácticas avanzadas

- **Diseña el mapa de contextos antes que las clases** — antes de escribir código, identifica qué contextos existen y cómo se relacionan (¿uno depende del otro?, ¿se traducen mutuamente?). Saltar directo a las entidades sin acordar las fronteras es la causa más común de que un "Cliente" acabe con quince campos que solo tienen sentido en la mitad de los casos.
- **Sospecha del modelo anémico** — si tus clases de dominio son solo propiedades públicas con getters y setters, y toda la lógica vive en clases de servicio externas, no tienes un modelo de dominio: tienes una base de datos con pasos extra. El comportamiento debe vivir junto a los datos que protege (ver [Entidades y Agregados](Entidades-y-Agregados.md)).
- **No repartas el lenguaje ubicuo entre contextos distintos** — es tentador reutilizar la misma clase `Cliente` en Catálogo y en Facturación "para no duplicar". Cada contexto necesita su propio modelo, aunque el nombre se parezca; forzar uno compartido acopla contextos que deberían poder evolucionar por separado.
- **Reserva el "diseño estratégico" completo para negocios realmente complejos** — técnicas como el *context mapping* formal o el *event storming* dan mucho valor en dominios grandes con varios equipos, pero son coste innecesario en una aplicación pequeña con un único contexto claro.

## Recursos didácticos

Si quieres profundizar más allá de esta introducción, el libro de referencia es *Domain-Driven Design* de Eric Evans (conocido como "el libro azul"); y la técnica de taller *EventStorming* (con post-its de colores modelando eventos de negocio) es una forma muy visual y amena de descubrir los contextos y eventos de un dominio en grupo.

---

*En resumen: DDD pone el modelo del negocio en el centro, compartiendo un lenguaje con las personas expertas y dividiendo dominios complejos en contextos delimitados, cada uno con su propio modelo consistente.*
