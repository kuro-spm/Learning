# Contrato primero

## ¿Qué es?

Un **contrato de API** es un documento —casi siempre un fichero YAML en formato **OpenAPI**— que describe qué rutas existen, qué recibe y qué devuelve cada una, y qué errores pueden aparecer, de una forma que entienden tanto las personas como las herramientas. **Contrato primero** (*design-first*) significa escribirlo y acordarlo **antes** de programar la implementación.

## ¿Por qué existe?

Sin contrato, la API es *lo que el código hace hoy*. Quien consume (otro equipo, una aplicación móvil, un cliente externo) lo averigua por ingeniería inversa: llama, mira qué vuelve y deduce. Y cuando el código cambia, nadie se entera hasta que algo se rompe en producción.

Imagina una tienda online con un equipo de frontend y otro de backend. Sin contrato, el flujo habitual es este: backend publica `GET /products/17`, frontend lo prueba con Postman, copia la forma de la respuesta a mano, y dos semanas después alguien renombra `price` a `unitPrice` «porque quedaba más claro». El frontend lo descubre cuando la ficha de producto enseña `NaN €`.

Con contrato primero, el cambio se propone **sobre el fichero**, se revisa antes de escribir una línea de código y ambos equipos trabajan contra el mismo texto.

> Si ya conoces las `interface` de C# o Java, piensa en el contrato como la `interface` entre dos sistemas: define qué se puede pedir y qué se obtiene, sin decir cómo se implementa. La diferencia es que viaja por la red, y por eso no hay compilador que avise de que alguien la ha roto.

## ¿Cuándo y para qué se usa?

- Cuando **más de un equipo** (o más de un cliente) depende de la misma API: una web, una app móvil, un socio que integra tu servicio.
- Cuando quieres **generar** cosas a partir de la API: clientes tipados, mocks, documentación navegable, validaciones.
- Cuando necesitas **decidir qué cambios son seguros**: sin un documento de referencia, «romper» es una opinión; con él, es una comparación.

El ejemplo conductor de toda la colección es una tienda online con `Product` y `Order`.

## Anatomía mínima de un contrato OpenAPI 3.1

Este contrato describe una sola operación: leer un producto.

```yaml
openapi: 3.1.0
info:
  title: Tienda — API de catálogo
  version: 1.0.0
paths:
  /products/{id}:
    get:
      operationId: getProduct
      summary: Obtener un producto por su identificador
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: string, format: uuid }
      responses:
        '200':
          description: El producto.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Product' }
        '404':
          description: No existe un producto con ese identificador.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }
components:
  schemas:
    Product:
      type: object
      required: [id, name, price]
      properties:
        id: { type: string, format: uuid }
        name: { type: string }
        price: { type: number, description: Precio unitario en euros, sin impuestos. }
    Error:
      type: object
      required: [code, message]
      properties:
        code: { type: string }
        message: { type: string }
```

Cuatro bloques, cada uno con su papel:

- **`info`** identifica el contrato y lleva su **versión**, que es la que cambia cuando el contrato cambia (se verá en [Versionado de contratos](Versionado-de-Contratos.md)).
- **`paths`** lista las rutas y, por cada una, los verbos con sus parámetros y todas las respuestas posibles, errores incluidos.
- **`operationId`** es el nombre estable de la operación. Los generadores lo usan para nombrar el método del cliente (`getProduct`), así que **cambiarlo rompe a quien lo use**, aunque la URL siga igual.
- **`components`** guarda los *schemas* reutilizables. Se enlazan con `$ref` para no repetir `Product` en cada respuesta.

Una respuesta que cumple ese contrato:

```json
{ "id": "3f2c...", "name": "Taza de cerámica", "price": 9.5 }
```

## El flujo: del cambio al código

El orden importa más que la herramienta:

1. Se propone el cambio **en el YAML**, en una rama o *pull request*.
2. Quienes consumen y quienes implementan lo revisan sobre el texto.
3. Se aprueba (quién y cómo se verá en [Gobierno de cambios en contratos](Gobierno-de-Cambios-en-Contratos.md)).
4. Se implementa el servidor y se regeneran los clientes.
5. Un test comprueba que la API real cumple lo que dice el contrato.

El camino contrario, **código primero** (*code-first*), consiste en escribir el servidor y dejar que el contrato se genere de las anotaciones:

| | Contrato primero | Código primero |
|---|---|---|
| El cambio se revisa… | antes de implementar | cuando ya está implementado |
| Los equipos pueden trabajar en paralelo | Sí, contra el mismo fichero | No: el contrato aparece al final |
| El contrato refleja… | lo que se **decidió** | lo que el código **hace** |
| Riesgo típico | El código se desvía del contrato | Un cambio accidental del código cambia el contrato sin que nadie lo decida |
| Encaja bien en… | APIs con varios consumidores | Prototipos y APIs internas de un único consumidor |

Con código primero, un refactor inocente (renombrar una propiedad de una clase) cambia la API pública sin que nadie lo haya decidido. Esa es la razón de fondo para preferir el contrato como fuente.

## Qué se obtiene del contrato

Un contrato bien mantenido deja de ser documentación y pasa a ser una **entrada** para otras herramientas:

| Herramienta | Qué hace con el contrato |
|---|---|
| Generador de clientes (por ejemplo `openapi-typescript`) | Crea los tipos y las llamadas del frontend, que se rompen al compilar si el contrato cambia |
| *Linter* (por ejemplo Spectral) | Comprueba reglas de estilo: que toda operación tenga `operationId`, que todo error tenga `code`… |
| Servidor *mock* | Responde con los ejemplos del contrato antes de que exista el backend |
| Comprobador de rupturas (por ejemplo `oasdiff`) | Compara dos versiones y avisa de si el cambio rompe a alguien |

## Buenas prácticas avanzadas

- **Cambia el contrato y el código en el mismo cambio.** Si viven en repositorios o ciclos distintos, antes o después se desfasan. El contrato en el mismo repositorio que el servidor, y una revisión que mire los dos a la vez, es lo que lo mantiene honesto.
- **Trata `operationId` como parte de la API.** Es el nombre del método en todos los clientes generados. Renombrarlo «por claridad» es una ruptura que el compilador del cliente detecta, pero que el servidor ni nota.
- **Divide por áreas, no por pantalla.** Un contrato por dominio (catálogo, pedidos, usuarios) se revisa y versiona por separado; uno gigante hace que cada cambio parezca afectar a todo. Evita también el extremo contrario: un fichero por endpoint pierde los `schemas` compartidos.
- **Reutiliza con `$ref` y no copies.** Si el mismo `Product` aparece pegado en cinco respuestas, un día solo se cambiará en cuatro.
- **Pon el *linter* en integración continua.** Las reglas de estilo que dependen de la memoria de quien revisa se olvidan; las que ejecuta una máquina no.
- **Escribe el contrato desde el punto de vista de quien consume.** Si un campo existe solo porque el servidor lo tiene a mano, pregúntate si alguien lo necesita. Cada campo publicado es un compromiso de mantenerlo.

## Documentación oficial

- [Especificación OpenAPI 3.1](https://spec.openapis.org/oas/v3.1.0) — la fuente normativa; consulta aquí qué significa exactamente cada palabra clave cuando la duda es de interpretación.
- [Learn OpenAPI](https://learn.openapis.org/) — la guía introductoria de la propia iniciativa OpenAPI; buen punto de partida para el primer contrato.
- [Spectral](https://github.com/stoplightio/spectral) — el *linter* de contratos más extendido; el README explica cómo escribir reglas propias.

## Recursos didácticos

- [Swagger Editor](https://editor.swagger.io/) — pega el YAML de arriba y verás la documentación generada al instante; sirve para experimentar sin instalar nada.

---

*En resumen: el contrato es el acuerdo escrito entre quien ofrece una API y quien la consume, y sale ganando quien lo escribe y lo revisa antes del código, no después.*
