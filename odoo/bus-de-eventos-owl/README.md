# Bus de eventos de OWL — Guía de tecnología

Cómo dos partes de la interfaz del cliente web de Odoo que no tienen relación entre sí —un widget de la barra de navegación y una vista ya abierta, por ejemplo— pueden avisarse de un cambio sin conocerse. Una sola ficha, centrada en `env.bus`: qué es, cómo se dispara y se escucha un evento, cómo engancharlo a una vista que no es tuya con `patch()`, y cuándo este mecanismo se queda corto y hace falta el bus real de Odoo (`bus.bus`, por WebSocket).

Está pensada para perfiles backend que ya programan a diario pero llegan nuevos al cliente web de Odoo (OWL): no presupone experiencia previa con el framework.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [El bus de eventos de OWL (env.bus)](Bus-de-Eventos-OWL.md) | Única ficha de la colección: el mecanismo completo, con ejemplo guiado de principio a fin. |

---

> Piezas relacionadas: [Tipos de notificación](../notificaciones/Tipos-de-Notificacion.md) sitúa el bus real de Odoo (`bus.bus`, por WebSocket) entre el resto de mecanismos con los que el sistema avisa a las personas — el apartado 8 es la referencia rápida para no confundirlo con `env.bus`. [Notificaciones de interfaz](../notificaciones/Notificaciones-de-Interfaz.md) usa los mismos *hooks* del cliente web (`useService`) que aparecen aquí.
