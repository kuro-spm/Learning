# Qué es una API

## ¿Qué es?

Una **API** (*Application Programming Interface*, interfaz de programación de aplicaciones) es un conjunto de reglas que permite que dos programas hablen entre sí. Define **qué** puedes pedir, **cómo** pedirlo y **qué** recibirás a cambio, sin que tengas que saber cómo funciona el otro programa por dentro.

## ¿Por qué existe?

Piensa en el camarero de un restaurante. Tú (un programa) no entras en la cocina (otro programa) a cocinar: le dices al camarero lo que quieres de una carta, él lo lleva a cocina y te trae el plato. La **carta** es el contrato (lo que puedes pedir), el **camarero** es la API (el intermediario) y la **cocina** es la implementación que no necesitas conocer.

Las APIs existen porque el software no vive aislado: una app de móvil necesita datos de un servidor, una web de pagos necesita hablar con el banco, un programa necesita usar funciones de otro. Sin un contrato claro, cada conexión sería un caos. La API pone orden: una puerta de entrada estable y documentada.

> Si ya programas, ya usas APIs sin pensarlo: cuando llamas a un método de una librería, esa librería te expone una API. Una *web API* es lo mismo, pero la llamada viaja por la red.

## ¿Cuándo y para qué se usa?

Aparece en cualquier sitio donde dos piezas de software se comuniquen:

- Una **app de tareas** que guarda tus notas en la nube llama a la API del servidor.
- Una **tienda online** consulta la API de una pasarela de pago para cobrar.
- Un **blog** usa la API del clima para mostrar la previsión en la cabecera.
- Un equipo interno expone una API para que el frontend y el backend se entiendan.

Este tutorial trata sobre los **tipos de API**: existen distintos "estilos" de construir esa puerta (REST, GraphQL, gRPC, WebSockets...), cada uno pensado para necesidades diferentes.

## Lo mínimo que necesitas saber

**1. Una API es un contrato, no una implementación**

Define entradas y salidas. Mientras el contrato no cambie, la cocina puede reescribirse entera sin que tú te enteres.

```http
GET /api/products/42
→ 200 OK
{ "id": 42, "name": "Teclado", "price": 29.90 }
```

**2. Casi siempre hay un cliente y un servidor**

El **cliente** pide, el **servidor** responde. El que pide no tiene por qué ser un humano: suele ser otro programa.

```csharp
// El cliente hace la petición; el servidor del otro lado responde
var response = await httpClient.GetAsync("https://api.tienda.com/products/42");
```

**3. Necesitan un formato de datos común**

Para entenderse, ambos lados acuerdan un formato. El más habitual hoy es **JSON** (texto legible con pares clave-valor); antes era común **XML**.

```json
{ "id": 42, "name": "Teclado", "inStock": true }
```

**4. Suelen requerir identificarse**

Muchas APIs piden una credencial (una *API key* o un *token*) para saber quién llama y si tiene permiso.

```http
GET /api/orders
Authorization: Bearer eyJhbGciOierf...
```

**5. No todas las APIs son web APIs**

La palabra API también describe las funciones que una librería ofrece a tu código. En este tutorial nos centramos en las que comunican sistemas **a través de la red**.

## Lo que NO hace

- **No es una base de datos** — es la puerta de acceso; los datos viven detrás de ella.
- **No es un lenguaje ni un protocolo concreto** — es un concepto; REST, GraphQL o gRPC son *formas* distintas de implementarlo.
- **No expone el funcionamiento interno** — ese es justo su objetivo: ocultar la cocina y ofrecer solo la carta.
- **No garantiza seguridad por sí sola** — proteger la puerta (autenticación, permisos) es trabajo aparte.

---

*En resumen: una API es el camarero entre dos programas —un contrato que dice qué puedes pedir y qué recibirás, sin que tengas que entrar en la cocina.*
