# Tipos de APIs — Guía de tecnología

Guía introductoria y **extensa** sobre los distintos tipos de APIs: qué es una API y, sobre todo, los muchos "estilos" para construir una (REST, GraphQL, gRPC, SOAP, WebSockets, webhooks, eventos...). Cada estilo resuelve un problema distinto; el objetivo es que sepas reconocerlos y elegir el adecuado.

Está escrita para perfiles junior con nociones básicas de programación, sin asumir experiencia previa con APIs. Los ejemplos son genéricos (una tienda online, un chat, una app de tareas) y, aunque muchos usan C#/JSON, las ideas valen para cualquier lenguaje.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Empezamos por el concepto, seguimos por los estilos de petición-respuesta (los más comunes), pasamos al tiempo real, luego a la comunicación asíncrona, y cerramos con dos formas de *clasificar* APIs que no dependen de la tecnología.

### 1. Fundamentos

Antes de los tipos, entiende qué es una API.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Qué es una API](Que-es-una-API.md) | El concepto base: el "camarero" entre dos programas. Sin esto, el resto no encaja. |

### 2. Estilos de petición-respuesta

El cliente pide y el servidor responde. Aquí viven los estilos más usados en la web.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [REST](REST.md) | El estándar de facto. Empieza por aquí: casi todo lo demás se compara con REST. |
| 3 | [GraphQL](GraphQL.md) | El cliente pide exactamente los datos que necesita. Nace de las limitaciones de REST. |
| 4 | [RPC](RPC.md) | Pensar en acciones (funciones remotas) en vez de en recursos. La raíz de gRPC y SOAP. |
| 5 | [gRPC](gRPC.md) | RPC moderno, binario y rápido, para que los microservicios se hablen entre sí. |
| 6 | [SOAP](SOAP.md) | El veterano formal basado en XML. Aún imprescindible en banca, seguros y *legacy*. |

### 3. Tiempo real

Cuando los datos llegan "en vivo" y el servidor necesita avisar sin que le pregunten.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 7 | [WebSockets](WebSockets.md) | Canal permanente y bidireccional. La base del tiempo real (chats, juegos). |
| 8 | [Server-Sent Events](Server-Sent-Events.md) | Flujo en una sola dirección (servidor → cliente). Más simple que WebSockets. |
| 9 | [Long Polling](Long-Polling.md) | El truco clásico para simular tiempo real con HTTP normal. Útil como respaldo. |

### 4. Comunicación asíncrona y por eventos

El servidor (u otro sistema) te avisa cuando ocurre algo, sin que estés preguntando.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 10 | [Webhooks](Webhooks.md) | Una "API al revés": el servicio te llama a ti cuando pasa un evento. |
| 11 | [APIs Dirigidas por Eventos](APIs-Dirigidas-por-Eventos.md) | Sistemas que publican eventos y otros reaccionan, desacoplados por un *broker*. |

### 5. Clasificar APIs (no por tecnología)

Dos formas de ordenar APIs que no dependen del estilo técnico.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 12 | [APIs por Audiencia](APIs-por-Audiencia.md) | Públicas, privadas y de socios: una clasificación por *quién* las usa. |
| 13 | [SDK](SDK.md) | No es un tipo de API, pero se confunde a menudo: la herramienta que envuelve una API. |

---

## Índice completo por categoría

<details>
<summary>Ver todos los archivos</summary>

**Fundamentos**
- [Qué es una API](Que-es-una-API.md)

**Estilos de petición-respuesta**
- [REST](REST.md)
- [GraphQL](GraphQL.md)
- [RPC](RPC.md)
- [gRPC](gRPC.md)
- [SOAP](SOAP.md)

**Tiempo real**
- [WebSockets](WebSockets.md)
- [Server-Sent Events](Server-Sent-Events.md)
- [Long Polling](Long-Polling.md)

**Comunicación asíncrona y por eventos**
- [Webhooks](Webhooks.md)
- [APIs Dirigidas por Eventos](APIs-Dirigidas-por-Eventos.md)

**Clasificación y herramientas relacionadas**
- [APIs por Audiencia](APIs-por-Audiencia.md)
- [SDK](SDK.md)

</details>

---

> ¿Te interesa cómo se construye el backend que sirve estas APIs? Echa un vistazo a la guía de [Clean Architecture](../clean-architecture/README.md) y a la de [transición de WPF a web](../de-wpf-a-web/README.md) si vienes de escritorio.
