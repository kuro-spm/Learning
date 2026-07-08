# Testing end-to-end (E2E) — Guía de tecnologías

Guía introductoria de los tests **end-to-end**: los que arrancan la aplicación entera y la manejan a través de un navegador real, como lo haría un usuario. Van en la cima de la pirámide de tests, por encima de los unitarios y los de integración, y protegen los flujos que no te puedes permitir romper.

Cada ficha explica qué es la herramienta, por qué existe, cuándo se usa y lo mínimo para no perderte — sin depender de ningún proyecto concreto.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Playwright](Playwright.md) | La herramienta E2E de referencia: automatiza un navegador real para probar login, checkout y demás flujos críticos de punta a punta. |

---

> Antes de saltar aquí conviene tener claro el mapa completo en [Tipos de tests](../testing-dotnet/Tipos-de-tests.md), y recordar que el E2E no sustituye a los tests de integración con base de datos real ([Testcontainers](../testing-dotnet/Testcontainers.md)) ni a los unitarios de frontend ([Vitest](../../desarrollo-web/frontend-react/Vitest.md)) — los complementa.
