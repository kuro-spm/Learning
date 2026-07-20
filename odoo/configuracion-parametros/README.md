# Configuración de parámetros en Odoo — Guía de tecnología

Colección introductoria sobre **las distintas maneras de configurar parámetros en Odoo**: dónde poner cada valor configurable para no dejarlo clavado en el código. Va desde la configuración del servidor hasta las referencias a registros concretos, pasando por los parámetros globales, la pantalla de Ajustes, la configuración por compañía y los valores por defecto. Pensada para perfiles junior que empiezan a desarrollar o administrar Odoo.

La idea que une toda la colección es una sola pregunta: **¿dónde vive este valor?** Elegir bien el sitio evita el problema más común en un módulo poco cuidado —números y textos "mágicos" repartidos por el código que se rompen al cambiar de entorno.

---

## Orden de lectura recomendado

De la capa más externa (el servidor) a la más interna (referencias entre registros). Cada ficha resuelve un tipo de valor distinto.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [El fichero odoo.conf](El-Fichero-odoo-conf.md) | La capa de infraestructura: lo que el servidor necesita para arrancar, antes de que exista lógica de negocio. |
| 2 | [Parámetros del sistema (ir.config_parameter)](Parametros-del-Sistema.md) | El almacén global clave-valor de la base de datos: valores que se cambian en caliente sin redeploy. |
| 3 | [Ajustes (res.config.settings)](Ajustes.md) | La pantalla amable que expone esos parámetros al usuario de negocio. |
| 4 | [Configuración por compañía](Configuracion-por-Compania.md) | El mismo parámetro con un valor distinto en cada empresa (multi-compañía). |
| 5 | [Valores por defecto](Valores-por-Defecto.md) | Qué se precarga en un campo al crear un registro, y desde dónde definirlo. |
| 6 | [Referencias por ID externo (XML ID)](Referencias-por-ID-Externo.md) | Cómo apuntar a un registro concreto sin clavar su ID numérico, que cambia entre bases de datos. |

---

> ¿Vas a aplicar estos cambios sobre un sistema real? Repasa antes cómo hacerlo sin riesgos en la guía de [pruebas seguras](../pruebas-seguras/README.md).
