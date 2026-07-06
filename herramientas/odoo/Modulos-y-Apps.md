# Módulos y Apps

**¿Qué es?** Un módulo (que en la interfaz de Odoo se llama *app*) es un paquete que añade una funcionalidad concreta al sistema: ventas, inventario, contabilidad, un informe nuevo, un campo extra... Odoo es, en el fondo, un núcleo pequeño más un montón de módulos que activas o desactivas según tus necesidades.

---

## ¿Por qué existe?

Nadie usa "todo Odoo": una tienda online no necesita el módulo de fabricación, y un taller quizá no necesita el de e-commerce. Dividir la funcionalidad en módulos permite instalar **solo lo que hace falta**, mantener el sistema ligero y añadir capacidades más adelante sin rehacer nada.

> Es la misma idea que instalar paquetes (`npm install`, `pip install`) para no reinventar la rueda: cada módulo es una pieza reutilizable que enchufas cuando la necesitas.

---

## ¿Cuándo y para qué se usa?

- Activar una función estándar: instalas el módulo **Inventario** y de repente tienes almacenes, movimientos de stock y albaranes.
- Añadir una integración: un módulo que conecta Odoo con una pasarela de pago o una empresa de mensajería.
- Personalizar tu instalación: un **módulo propio** (desarrollado a medida) que añade un campo, una pantalla o una regla de negocio específica de tu empresa.

---

## Lo mínimo que necesitas saber

**1. Tres orígenes de módulos**

- **Oficiales de Odoo** — vienen con el sistema (ventas, compras, contabilidad...).
- **De terceros** — publicados en la Odoo Apps Store por otras empresas.
- **A medida** — desarrollados específicamente para una organización.

**2. Los módulos a medida viven en carpetas de *addons***

Un módulo personalizado es una carpeta con una estructura fija. El servidor la carga si está en su ruta de *addons*.

```
mi_modulo/
├── __manifest__.py   # nombre, versión, dependencias del módulo
├── __init__.py
├── models/           # la lógica (Python)
└── views/            # las pantallas (XML)
```

**3. Instalar un módulo es un cambio serio**

Instalar o actualizar un módulo modifica la base de datos (crea tablas, columnas, datos). No es como abrir una pantalla: es una operación que **conviene probar antes en una copia**, nunca directamente sobre el sistema en producción.

---

## Lo que NO hace

- **Un módulo no es un simple ajuste** — instalarlo puede alterar el esquema de la base de datos de forma difícil de revertir.
- **No todos los módulos de terceros son fiables** — conviene revisar su origen y probarlos en un entorno aislado antes de confiarles datos reales.
- **Desinstalar no siempre deja todo como estaba** — puede borrar datos asociados; por eso se prueba primero en una copia.

---

## Buenas prácticas avanzadas

- **Hereda, no edites el módulo original** — para cambiar algo de un módulo estándar se crea un módulo propio que lo extiende (`_inherit` en los modelos, vistas heredadas en el XML). Si editas los ficheros del módulo original, la siguiente actualización de Odoo pisará tus cambios sin avisar.
- **Ensaya también la desinstalación** — antes de llevar un módulo a producción, prueba en la copia no solo instalarlo, sino quitarlo, y mira qué datos desaparecen. Así sabes si "desinstalar" es una marcha atrás real o si tu única salida sería restaurar un backup.
- **Un módulo de terceros te compromete a su mantenimiento** — cada rama de Odoo (16.0, 17.0...) necesita su propia versión del módulo. Antes de adoptarlo, comprueba que su autor lo migra activamente a las versiones nuevas; si lo abandona, portar ese código te tocará a ti o bloqueará tu actualización de Odoo.

---

*En resumen: los módulos son las piezas de Lego con las que se arma un Odoo a medida; instalarlos o quitarlos toca la base de datos, así que siempre se prueban antes en una copia.*
