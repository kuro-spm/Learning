# Fundamentos de Odoo — Guía de tecnología

Colección introductoria para **entender qué es Odoo y cómo está montado por dentro** antes de tocar nada. Pensada para perfiles junior que se acercan por primera vez a este ERP: qué problema resuelve, cómo se le añade funcionalidad y qué conceptos propios (los modelos de "persona", el modo desarrollador) hay que tener claros para no perderse.

Es la base de todo lo demás de la carpeta: sin esto, ni las pruebas seguras ni el desarrollo de búsquedas y filtros tienen contexto.

---

## Orden de lectura recomendado

Cada ficha apoya a la siguiente: primero qué es Odoo, luego cómo crece, después su modelo de personas y, por último, la vista técnica.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Odoo](Odoo.md) | El punto de partida: qué es, para qué sirve y cómo encaja Python + PostgreSQL. |
| 2 | [Módulos y Apps](Modulos-y-Apps.md) | Cómo se añade funcionalidad; por qué instalar un módulo es un cambio serio. |
| 3 | [Partner, Usuario y Empleado](Partner-Usuario-y-Empleado.md) | Los tres modelos de "persona" de Odoo y por qué no son lo mismo: contacto vs. login vs. ficha laboral. |
| 4 | [Modo desarrollador](Modo-Desarrollador.md) | El interruptor que revela la capa técnica y por qué hay que usarlo con cuidado. |

---

> ¿Listo para tocar un sistema real? Sigue con [Pruebas seguras](../pruebas-seguras/README.md) antes de probar nada sobre datos de producción.
