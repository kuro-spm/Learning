# Modo Desarrollador

**¿Qué es?** El modo desarrollador (*developer mode* o *debug mode*) es un ajuste de Odoo que desbloquea opciones técnicas ocultas: ver los nombres internos de los campos, editar vistas, acceder a menús de configuración avanzada, importar/exportar datos y muchas herramientas de diagnóstico. No cambia nada por sí mismo; solo **muestra** lo que normalmente está escondido.

---

## ¿Por qué existe?

La interfaz normal de Odoo está pensada para usuarias de negocio, así que oculta lo técnico para no abrumar ni provocar errores. Pero quien administra o personaliza el sistema necesita ver "las tripas". El modo desarrollador es el interruptor que revela esa capa técnica sin tener que tocar código.

> Piensa en él como las "herramientas de desarrollador" del navegador (F12): siempre están ahí, pero solo aparecen cuando las activas, y con ellas puedes ver y tocar cosas que un usuario normal no ve.

---

## ¿Cuándo y para qué se usa?

- Averiguar el **nombre técnico** de un campo o modelo (por ejemplo, para configurar un informe o una automatización).
- Editar **vistas** y menús, o revisar la configuración avanzada de un módulo.
- Diagnosticar un problema: ver qué acción se ejecuta detrás de un botón, revisar registros de correo, etc.

---

## Lo mínimo que necesitas saber

**1. Cómo se activa**

Desde *Ajustes*, al final de la página, con el enlace **"Activar el modo desarrollador"**. O directamente por URL:

```
https://miempresa.odoo.com/web#action=base_setup.action_general_configuration
https://miempresa.odoo.com/web?debug=1        # activarlo
https://miempresa.odoo.com/web?debug=0        # desactivarlo
```

**2. Solo revela, pero lo que revela es peligroso**

Activarlo no rompe nada. El riesgo está en **lo que puedes hacer una vez activado**: borrar campos, editar vistas, ejecutar acciones masivas o modificar datos técnicos. Con más poder, más facilidad para estropear algo.

**3. En producción, actívalo con cuidado y desactívalo al terminar**

En un sistema real, úsalo solo para consultar o para cambios que sabes que son seguros. Evita las opciones destructivas (como el botón de borrar un campo o ejecutar métodos del servidor) salvo que estés en una copia de pruebas.

---

## Lo que NO hace

- **No es un entorno de pruebas** — sigues sobre la misma base de datos real; lo que toques aquí afecta a los datos de verdad.
- **No protege de errores** — al contrario, expone acciones que pueden dañar la instalación si te equivocas.
- **No requiere saber programar** — pero muchas de sus opciones asumen que entiendes lo que estás tocando.

---

## Buenas prácticas avanzadas

- **Hay más de un modo: `?debug=assets`** — esta variante sirve el JavaScript y el CSS sin minificar, imprescindible para depurar errores de interfaz con las herramientas del navegador: los *stack traces* pasan a señalar ficheros legibles en lugar de un *bundle* comprimido.
- **Es un ajuste por sesión, no del sistema** — activarlo solo te afecta a ti: no "enciendes" nada para el resto de usuarios ni puedes apagárselo a otra persona desde tu sesión. Cuidado con las extensiones de navegador que lo mantienen siempre activo: normalizan trabajar en producción con las tijeras afiladas en la mano.
- **El superusuario es para diagnosticar permisos, no para trabajar** — con el modo desarrollador puedes convertirte en superusuario (*Become Superuser*), que ignora todas las reglas de acceso. Es la forma rápida de distinguir "es un bug" de "es un problema de permisos": si como superusuario funciona, el problema son los permisos. Pero todo lo que edites así se salta las protecciones normales, así que vuelve a tu usuario en cuanto termines.

---

*En resumen: el modo desarrollador enciende la luz sobre la capa técnica de Odoo; no cambia nada solo, pero te da tijeras afiladas, así que en producción se usa con guantes y se apaga al acabar.*
