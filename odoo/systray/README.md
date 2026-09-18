# Systray de Odoo — Guía de tecnología

Cómo funciona la franja de iconos superior derecha del backend de Odoo (notificaciones, fichaje, indicadores propios de cada usuario) y cómo se construye un icono propio con un componente Owl: el registro `systray`, de dónde vienen sus datos, su estado reactivo y cómo se engancha al *bundle* de assets del backend. Pensada para quien ya sabe programar pero no ha tocado el frontend de Odoo.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Systray](Systray.md) | La guía completa: qué es, el registro, el componente Owl, de dónde salen los datos y cómo se registra en el manifest. |

---

> Si lo que buscas es cómo Odoo avisa a las personas con avisos flotantes, diálogos o correos —no un icono permanente en la barra—, esa es la colección de [notificaciones](../notificaciones/README.md). Si el icono del systray necesita avisar a una vista ya abierta de que algo cambió (y viceversa), ese mecanismo es el [bus de eventos de OWL](../bus-de-eventos-owl/README.md).
