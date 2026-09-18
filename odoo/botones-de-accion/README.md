# Botones de acción — Guía de tecnología

Cómo añadir una acción propia a una vista de Odoo sin construir un formulario ni un asistente: el botón `type="object"`, la herencia de vistas con `xpath` que hace falta para insertarlo, y las condiciones que deciden cuándo se ve.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Botones de acción en Kanban](Botones-de-Accion-en-Kanban.md) | El recorrido completo: heredar una vista con `xpath`, declarar los campos que hacen falta, la anatomía del botón y qué hace Odoo con lo que el método devuelve. |

---

> Si el botón que buscas dispara un aviso al terminar, la parte de "qué mostrar" está en la guía de [notificaciones](../notificaciones/README.md). Si el cambio que hace el botón tiene que reflejarse en otra parte de la pantalla que ya estaba abierta (un icono del [systray](../systray/README.md), por ejemplo), ese mecanismo es el [bus de eventos de OWL](../bus-de-eventos-owl/README.md).
