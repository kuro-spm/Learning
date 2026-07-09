# Odoo — Guía de tecnología

Colección introductoria sobre **Odoo** (el ERP de código abierto) con foco en algo muy concreto: **cómo hacer pruebas de forma segura sobre un Odoo que está en producción**. Está escrita para perfiles junior, sin dar por supuesta experiencia previa con Odoo ni con sistemas de gestión.

Primero entenderás qué es Odoo y cómo se organiza; luego, el conjunto de buenas prácticas para tocar un sistema real sin llevártelo por delante.

---

## Orden de lectura recomendado

### 1. Entender Odoo

Qué es y cómo está montado por dentro. Necesario para que todo lo demás tenga sentido.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Odoo](Odoo.md) | El punto de partida: qué es, para qué sirve y cómo encaja Python + PostgreSQL. |
| 2 | [Módulos y Apps](Modulos-y-Apps.md) | Cómo se añade funcionalidad; por qué instalar un módulo es un cambio serio. |
| 3 | [Modo desarrollador](Modo-Desarrollador.md) | El interruptor que revela la capa técnica y por qué hay que usarlo con cuidado. |

### 2. Probar sin romper producción

El corazón de la guía: dónde probar, cómo aislar los datos reales y cómo tener siempre marcha atrás.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 4 | [Entornos: desarrollo, staging y producción](Entornos-dev-staging-produccion.md) | La idea base: separar entornos para tener un sitio donde equivocarte. |
| 5 | [Duplicar la base de datos](Duplicar-Base-de-Datos.md) | El primer paso: una copia realista de producción para probar sobre datos de verdad. |
| 6 | [Neutralizar la base de datos](Neutralizar-Base-de-Datos.md) | El paso imprescindible: apagar en la copia todo lo que puede escaparse al mundo real. |
| 7 | [Backups y restauración](Backups-y-Restauracion.md) | Tu red de seguridad: cómo volver atrás cuando algo sale mal. |
| 8 | [Pruebas seguras sobre un Odoo en producción](Pruebas-Seguras-en-Produccion.md) | La síntesis: el checklist de buenas prácticas que reúne todo lo anterior. |

---

> ¿Te interesa cómo un sistema separa los datos de varios clientes en una misma instalación? Échale un ojo a la guía de [multi-tenancy](../../arquitectura-de-software/multi-tenancy/README.md).
