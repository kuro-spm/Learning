# Arquitectura de software — Guías

Cómo organizar el código de una aplicación y cómo diseñar la comunicación entre sistemas: patrones de arquitectura, aislamiento multi-cliente y los distintos estilos de API.

---

## Contenido

### [Clean Architecture](clean-architecture/README.md)
Cómo separar dominio, casos de uso e infraestructura para que el código sea fácil de mantener, probar y cambiar.

### [Patrones de diseño](patrones-de-diseno/README.md)
Domain-Driven Design (bounded context, objetos de valor, entidades y agregados, repositorios, eventos de dominio) y una selección de patrones de diseño clásicos (Factory, Strategy, Decorator).

### [Multi-Tenancy](multi-tenancy/README.md)
Cómo una sola aplicación da servicio a muchos clientes (*tenants*) manteniendo sus datos aislados.

### [Tipos de APIs](tipos-de-apis/README.md)
Los distintos estilos para construir una API (REST, GraphQL, gRPC, SOAP, WebSockets, webhooks, eventos...) y cuándo elegir cada uno.
