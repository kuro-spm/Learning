# Odoo — Guía de tecnología

Colección introductoria sobre **Odoo** (el ERP de código abierto), organizada en cuatro subcolecciones: los **fundamentos** para entender qué es y cómo está montado, las **pruebas seguras** para tocar un sistema en producción sin romperlo, el **desarrollo de búsquedas y filtros** que ve el usuario, y las **maneras de configurar parámetros** para no dejar valores clavados en el código. Está escrita para perfiles junior, sin dar por supuesta experiencia previa con Odoo ni con sistemas de gestión.

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

### 4. [Configuración de parámetros](configuracion-parametros/README.md)

Las distintas maneras de guardar un valor configurable en Odoo, para no dejarlo clavado en el código.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 14 | [El fichero odoo.conf](configuracion-parametros/El-Fichero-odoo-conf.md) | La capa de infraestructura: lo que el servidor necesita para arrancar. |
| 15 | [Parámetros del sistema (ir.config_parameter)](configuracion-parametros/Parametros-del-Sistema.md) | El almacén global clave-valor de la base de datos, editable en caliente. |
| 16 | [Ajustes (res.config.settings)](configuracion-parametros/Ajustes.md) | La pantalla amable que expone esos parámetros al usuario de negocio. |
| 17 | [Configuración por compañía](configuracion-parametros/Configuracion-por-Compania.md) | El mismo parámetro con un valor distinto en cada empresa. |
| 18 | [Valores por defecto](configuracion-parametros/Valores-por-Defecto.md) | Qué se precarga en un campo al crear un registro, y desde dónde definirlo. |
| 19 | [Referencias por ID externo (XML ID)](configuracion-parametros/Referencias-por-ID-Externo.md) | Apuntar a un registro concreto sin clavar su ID numérico. |

---

> ¿Te interesa cómo un sistema separa los datos de varios clientes en una misma instalación? Échale un ojo a la guía de [multi-tenancy](../arquitectura-de-software/multi-tenancy/README.md).
