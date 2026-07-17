# Odoo — Guía de tecnología

Colección introductoria sobre **Odoo** (el ERP de código abierto), organizada en tres subcolecciones: los **fundamentos** para entender qué es y cómo está montado, las **pruebas seguras** para tocar un sistema en producción sin romperlo, y el **desarrollo de búsquedas y filtros** que ve el usuario. Está escrita para perfiles junior, sin dar por supuesta experiencia previa con Odoo ni con sistemas de gestión.

Cada subcolección tiene su propio índice con orden de lectura. Si empiezas de cero, recórrelas en el orden de abajo.

---

## Subcolecciones

### 1. [Fundamentos](fundamentos/README.md)

Qué es Odoo y cómo está montado por dentro. Necesario para que todo lo demás tenga sentido.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Odoo](fundamentos/Odoo.md) | El punto de partida: qué es, para qué sirve y cómo encaja Python + PostgreSQL. |
| 2 | [Módulos y Apps](fundamentos/Modulos-y-Apps.md) | Cómo se añade funcionalidad; por qué instalar un módulo es un cambio serio. |
| 3 | [Partner, Usuario y Empleado](fundamentos/Partner-Usuario-y-Empleado.md) | Los tres modelos de "persona" de Odoo y por qué no son lo mismo: contacto vs. login vs. ficha laboral. |
| 4 | [Modo desarrollador](fundamentos/Modo-Desarrollador.md) | El interruptor que revela la capa técnica y por qué hay que usarlo con cuidado. |

### 2. [Pruebas seguras](pruebas-seguras/README.md)

El corazón de la guía: dónde probar, cómo aislar los datos reales y cómo tener siempre marcha atrás.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 5 | [Entornos: desarrollo, staging y producción](pruebas-seguras/Entornos-dev-staging-produccion.md) | La idea base: separar entornos para tener un sitio donde equivocarte. |
| 6 | [Duplicar la base de datos](pruebas-seguras/Duplicar-Base-de-Datos.md) | El primer paso: una copia realista de producción para probar sobre datos de verdad. |
| 7 | [Neutralizar la base de datos](pruebas-seguras/Neutralizar-Base-de-Datos.md) | El paso imprescindible: apagar en la copia todo lo que puede escaparse al mundo real. |
| 8 | [Backups y restauración](pruebas-seguras/Backups-y-Restauracion.md) | Tu red de seguridad: cómo volver atrás cuando algo sale mal. |
| 9 | [Pruebas seguras sobre un Odoo en producción](pruebas-seguras/Pruebas-Seguras-en-Produccion.md) | La síntesis: el checklist de buenas prácticas que reúne todo lo anterior. |

### 3. [Búsqueda y filtros](busqueda-y-filtros/README.md)

Ya en clave de desarrollo, cómo se construyen las herramientas de búsqueda y filtrado de un listado.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 10 | [Dominios](busqueda-y-filtros/Dominios.md) | El cimiento: el lenguaje con el que Odoo expresa cualquier filtro. |
| 11 | [Vista de búsqueda](busqueda-y-filtros/Vista-de-Busqueda.md) | Dónde se declaran los filtros, agrupaciones y campos buscables de un listado. |
| 12 | [Search Panel](busqueda-y-filtros/Search-Panel.md) | El panel lateral de filtrado por facetas: el tema estrella y el más visual. |
| 13 | [Dominios dinámicos](busqueda-y-filtros/Dominios-Dinamicos.md) | Filtros que se calculan según el contexto: otro campo, el usuario o una lógica en Python. |

---

> ¿Te interesa cómo un sistema separa los datos de varios clientes en una misma instalación? Échale un ojo a la guía de [multi-tenancy](../arquitectura-de-software/multi-tenancy/README.md).
