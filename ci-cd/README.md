# CI/CD — Guía de tecnologías

Documentación introductoria sobre **CI/CD** (Integración y Despliegue Continuo): qué es, cómo se describe un pipeline y qué herramientas y técnicas se usan para llevar código a producción de forma automática y segura.

Está escrita para perfiles junior con nociones básicas de programación y de Git, pero sin experiencia previa en automatización ni despliegues. Cada ficha explica qué es la tecnología o el concepto, por qué existe, cuándo se usa y lo mínimo que necesitas saber para no perderte.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Está pensado para que cada concepto apoye al siguiente.

### 1. Conceptos fundamentales

Empieza por entender la idea antes de mirar herramientas concretas.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [CI/CD](CI-CD.md) | La idea central. Qué significan las siglas y qué problema resuelven. Imprescindible antes de todo lo demás. |
| 2 | [Pipeline](Pipeline.md) | Cómo se describe el proceso paso a paso. Es el vocabulario común a todas las herramientas. |

### 2. Herramientas de CI/CD

Las plataformas concretas que ejecutan los pipelines. Lee al menos la primera; las demás según lo que uses.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [GitHub Actions](GitHub-Actions.md) | La opción más extendida si tu código está en GitHub. Buen punto de partida práctico. |
| 4 | [GitLab CI/CD](GitLab-CI.md) | El equivalente para repositorios en GitLab. Mismos conceptos, vocabulario propio. |
| 5 | [Jenkins](Jenkins.md) | El veterano autoalojado. Útil para entender el enfoque "servidor propio". |

### 3. Piezas que aparecen en casi cualquier pipeline

Conceptos transversales que usarás sea cual sea la herramienta.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 6 | [Docker en CI/CD](Docker-en-CI-CD.md) | Cómo se consigue un entorno reproducible y cómo se empaqueta la app para desplegarla. |
| 7 | [Secrets y variables de entorno](Secrets-y-Variables.md) | Cómo manejar contraseñas y tokens sin meterlos en el repositorio. Seguridad básica imprescindible. |
| 8 | [Estrategias de despliegue](Estrategias-de-Despliegue.md) | Las distintas formas de poner una versión en producción sin cortes y con vuelta atrás. |

### 4. Técnicas que verás en pipelines reales

Patrones concretos que aparecen al montar el CI/CD de un proyecto de verdad. Requieren los bloques anteriores.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 9 | [Git hooks (validación local)](Git-Hooks.md) | La primera línea de defensa: validar en tu máquina antes de gastar un minuto de CI. |
| 10 | [Monorepo y path-filters](Monorepo-y-Path-Filters.md) | Cómo evitar que un repo con varias áreas lo recompile todo en cada cambio. |
| 11 | [Entornos efímeros en CI](Entornos-Efimeros-en-CI.md) | Probar la aplicación completa sin mantener un servidor de pruebas permanente. |
| 12 | [Release por tag](Release-por-Tag.md) | Separar "integrar código" de "publicar producto": el deploy como decisión explícita. |

---

> ¿Quieres ver cómo encaja esto con el resto del stack? Echa un vistazo a las guías de [tecnologías backend y frontend](../technologies/) de esta misma colección de aprendizaje.
