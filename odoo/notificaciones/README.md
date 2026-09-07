# Notificaciones en Odoo — Guía de tecnología

Cómo avisa Odoo a las personas: el aviso flotante de la esquina, el diálogo que impide guardar, el historial del *chatter*, el correo con plantilla y la tarea con fecha límite. Una ficha por mecanismo, con qué API lo dispara, qué ve quien lo recibe y cuándo es el mecanismo equivocado.

Está pensada para perfiles backend que ya programan a diario pero llegan nuevos a Odoo: no presupone experiencia con el framework, y define los términos propios (*chatter*, *mixin*, subtipo, seguidor, actividad) antes de usarlos.

Todas las fichas usan el mismo ejemplo de principio a fin —una tienda online con un modelo `shop.order`, el pedido **SO0042** del cliente Marina Costa y el comercial Jordi Vidal— para que las piezas encajen entre documentos.

---

## Orden de lectura recomendado

Empieza por el mapa: es el que decide qué ficha necesitas. Las demás se apoyan en él y pueden leerse en cualquier orden, aunque el de la tabla va de lo más simple a lo más elaborado.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Tipos de notificación](Tipos-de-Notificacion.md) | El mapa: los diez mecanismos, los cuatro ejes que los separan y la tabla de decisión. Empieza aquí. |
| 2 | [Notificaciones de interfaz](Notificaciones-de-Interfaz.md) | El aviso flotante: `display_notification` desde Python y el servicio `notification` desde JavaScript. |
| 3 | [Errores y avisos bloqueantes](Errores-y-Avisos.md) | Las excepciones que el cliente sabe presentar, y por qué un `raise` deshace la transacción entera. |
| 4 | [Chatter, mensajes y seguidores](Chatter-y-Seguidores.md) | El historial del registro: `message_post`, subtipos, seguidores y rastreo de campos. La ficha central. |
| 5 | [Plantillas de correo](Plantillas-de-Correo.md) | `mail.template`: renderizado, cola de envío, adjuntos e idioma del destinatario. |
| 6 | [Actividades](Actividades.md) | El aviso que además es trabajo: `activity_schedule`, tipos, cierre y cadenas. |

---

## Pendiente de escribir

La colección está incompleta a propósito: el trabajo se detuvo el **2026-09-07** con seis fichas terminadas. Estos tres mecanismos están descritos y con ejemplo en la ficha 1 ([Tipos de notificación](Tipos-de-Notificacion.md), apartados 8, 9 y 10), pero **no tienen ficha propia todavía**:

| Ficha prevista | Qué debe cubrir |
|---|---|
| Tiempo real con el bus | `bus.bus`, el *mixin* `bus.listener.mixin` y `_bus_send`, el tipo `simple_notification`, canales propios con `_build_bus_channel_list`, la suscripción desde JavaScript y la semántica transaccional (los mensajes salen en el *post-commit*). |
| Notificaciones push del navegador | Push web con `mail.push` y `mail.push.device`, claves VAPID, *service worker*, el permiso del navegador y el cron de envío. |
| Notificaciones sin código | Acciones automatizadas (`base_automation`) y acciones de servidor de tipo `mail_post`, `next_activity`, `followers` y `webhook`, para configurar avisos desde la interfaz. |

Al añadirlas hay que hacer tres cosas más: enlazarlas en la tabla de orden de lectura de este índice, devolver los enlaces en la ficha 1 (donde ahora dice "Ficha propia pendiente de escribir" y "Pendiente" en la tabla de decisión) y restaurar los enlaces al bus que quedaron como texto plano en [Notificaciones de interfaz](Notificaciones-de-Interfaz.md) y [Actividades](Actividades.md).

---

> Piezas relacionadas en esta misma carpeta: [Fundamentos de Odoo](../fundamentos/README.md) para entender los modelos y el modo desarrollador que estas fichas dan por sabidos, [Configuración de parámetros](../configuracion-parametros/README.md) para no dejar clavados en el código los valores que un aviso necesita, y [Pruebas seguras](../pruebas-seguras/README.md) —imprescindible antes de probar envíos de correo sobre una copia de producción.
