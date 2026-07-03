# Clean Architecture — Guía de conceptos

Documentación introductoria sobre **Clean Architecture**, escrita para perfiles junior con nociones básicas de programación que quieren entender cómo organizar el código de una aplicación de negocio para que sea fácil de mantener, probar y cambiar.

Cada archivo explica qué es un concepto, por qué existe, cuándo se usa y lo mínimo que necesitas saber para no perderte. Los ejemplos son genéricos (una tienda online, un blog, una app de reservas) y no dependen de ningún proyecto concreto.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero. Primero la idea general y sus reglas; después las capas, de dentro hacia fuera; por último, cómo repartir el código cuando la aplicación se organiza en módulos.

### 1. La idea y sus reglas

Entiende primero qué propone Clean Architecture y los dos principios que la sostienen.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Clean Architecture](Clean-Architecture.md) | El mapa general: las capas y la idea de proteger el negocio. Empieza por aquí. |
| 2 | [Regla de Dependencia](Regla-de-Dependencia.md) | El principio central: las dependencias solo apuntan hacia dentro. |
| 3 | [Inversión de Dependencias](Inversion-de-Dependencias.md) | El mecanismo que permite cumplir la regla cuando el centro necesita algo externo. |

### 2. Las capas, de dentro hacia fuera

Recorre las capas en el mismo orden en que conviene diseñarlas: del corazón estable al borde cambiante.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 4 | [Capa de Dominio](Capa-de-Dominio.md) | El centro: entidades y reglas de negocio puras. La capa más importante. |
| 5 | [Casos de Uso](Casos-de-Uso.md) | Cómo se orquesta cada acción de la aplicación alrededor del dominio. |
| 6 | [Puertos y Adaptadores](Puertos-y-Adaptadores.md) | El vocabulario de las fronteras: cómo entra y sale la información del centro. |
| 7 | [Capa de Infraestructura](Capa-de-Infraestructura.md) | El borde técnico: base de datos, web y demás detalles reemplazables. |

### 3. Organizar el código a lo grande

Con las capas claras, queda la pregunta del día a día en un monolito modular: dónde va cada pieza nueva.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 8 | [Módulos vs Shared](Modulos-vs-Shared.md) | El criterio para decidir si el código nuevo va a un módulo propio o a los proyectos compartidos. |

---

> ¿Te interesa cómo encaja esto con tecnologías concretas? Echa un vistazo a la carpeta [`technologies/`](../technologies/), donde se documenta un stack real de backend y frontend.
