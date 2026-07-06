# Patrones de diseño — Guía de conceptos

> 🧭 Colección añadida como recomendación proactiva, no por una necesidad concreta de ningún proyecto.

Documentación introductoria sobre patrones de diseño habituales al modelar el dominio de una aplicación: primero las ideas y piezas de **Domain-Driven Design** (DDD), y después una selección de patrones de diseño clásicos (GoF) que aparecen a menudo en ese mismo contexto. Los ejemplos son genéricos (una tienda online, un sistema de pedidos) y no dependen de ningún proyecto concreto.

Esta colección complementa a [Clean Architecture](../clean-architecture/README.md): aquella explica **en qué capa** vive cada cosa y por qué las dependencias solo apuntan hacia dentro; esta explica **con qué patrones** se modela el contenido de esas capas, sobre todo el de la capa de dominio.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Primero la visión general de DDD y sus piezas de modelado; después, patrones de diseño clásicos que se combinan bien con ellas.

### 1. Domain-Driven Design: la visión general y sus piezas

Entiende primero el enfoque general y los bloques con los que se construye un modelo de dominio rico.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Domain-Driven Design](Domain-Driven-Design.md) | El mapa general: lenguaje ubicuo, bounded context y cómo encaja con Clean Architecture. Empieza por aquí. |
| 2 | [Value Objects](Value-Objects.md) | La pieza más pequeña: datos inmutables definidos por su valor, no por una identidad. |
| 3 | [Entidades y Agregados](Entidades-y-Agregados.md) | La pieza con identidad, y cómo se agrupan varias bajo una raíz que protege sus reglas conjuntas. |
| 4 | [Repositorios](Repositorios.md) | Cómo el dominio carga y guarda sus agregados sin conocer la tecnología de persistencia. |
| 5 | [Domain Events](Domain-Events.md) | Cómo un agregado comunica que algo ha pasado, sin acoplarse a quién reacciona. |

### 2. Patrones de diseño clásicos (GoF)

Patrones de propósito general que aparecen constantemente al construir el dominio y sus alrededores.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 6 | [Factory](Factory.md) | Cómo centralizar y proteger la creación de objetos con reglas propias. |
| 7 | [Strategy](Strategy.md) | Cómo intercambiar un algoritmo (cálculo, validación, ordenación) sin `switch` gigantes. |
| 8 | [Decorator](Decorator.md) | Cómo añadir capacidades (caché, logging, reintentos) envolviendo un objeto sin tocar su implementación. |

---

> Si no conoces todavía cómo se organizan las capas de una aplicación, empieza por [Clean Architecture](../clean-architecture/README.md) antes de esta colección: allí se explican la [Capa de Dominio](../clean-architecture/Capa-de-Dominio.md), los [Casos de Uso](../clean-architecture/Casos-de-Uso.md) y los [Puertos y Adaptadores](../clean-architecture/Puertos-y-Adaptadores.md) que estas fichas dan por conocidos.
