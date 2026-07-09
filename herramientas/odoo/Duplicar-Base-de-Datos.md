# Duplicar la base de datos

## ¿Qué es?

Es hacer una **copia completa** de la base de datos de producción —datos y adjuntos— para trabajar sobre ella en lugar de sobre el original. El resultado es un gemelo con datos realistas donde puedes probar sin tocar el sistema que usa la empresa cada día.

## ¿Por qué existe?

Probar sobre datos inventados no siempre revela los problemas: muchos errores solo aparecen con la configuración y el volumen reales (un cliente con miles de facturas, una regla contable concreta, un módulo instalado hace años). Por eso, para probar de verdad, se parte de una **copia de los datos reales**.

Duplicar da ese realismo. Pero una copia recién hecha **todavía no es segura**: hereda todas las conexiones de producción y, si la arrancas tal cual, podría enviar correos o disparar cobros reales. Por eso duplicar es solo la mitad del trabajo; la otra mitad es [neutralizar la copia](Neutralizar-Base-de-Datos.md) antes de usarla.

> Piensa en un simulador de vuelo: primero necesitas una réplica fiel del avión de verdad (eso es duplicar); después le quitas la conexión con el mundo real para que estrellarlo no tenga consecuencias (eso es neutralizar).

## ¿Cuándo y para qué se usa?

- Antes de **instalar o actualizar un módulo**, para ver el efecto sobre datos reales.
- Para **ensayar una migración** a una versión nueva de Odoo.
- Para **reproducir un error** que solo ocurre con los datos de producción.
- Para **formar** o hacer **demos** con información realista.

## Lo mínimo que necesitas saber

**1. Duplicar desde el gestor de bases de datos**

En el gestor de Odoo (`/web/database/manager`) hay un botón **Duplicate** que clona la base de datos y su filestore de una vez. Es la vía más cómoda cuando tienes acceso a esa pantalla.

**2. Duplicar por línea de comandos (restaurando un backup)**

Cuando montas el entorno tú, se hace restaurando una copia de seguridad con otro nombre:

```bash
# Restaurar una copia de producción con un nombre distinto
createdb odoo_test
pg_restore -d odoo_test produccion.dump
# y copiar también el filestore (adjuntos, imágenes) a la carpeta de la nueva BD
cp -r /var/lib/odoo/filestore/produccion /var/lib/odoo/filestore/odoo_test
```

**3. La base de datos son dos piezas: datos + filestore**

Odoo guarda las tablas en PostgreSQL, pero los adjuntos e imágenes viven en disco, en el **filestore**. Una copia solo de la base de datos, sin el filestore, arranca pero pierde documentos e imágenes. Copia siempre ambas cosas.

**4. La copia no es segura hasta neutralizarla**

Nada más duplicar, la copia sigue apuntando a los servidores de correo, pasarelas de pago e integraciones de producción. **No la arranques todavía**: el siguiente paso obligatorio es [neutralizarla](Neutralizar-Base-de-Datos.md).

## Lo que NO hace

- **No neutraliza** — duplicar solo copia; desactivar los efectos hacia el exterior es un paso aparte.
- **No se actualiza sola** — es una foto del momento; los datos nuevos de producción no aparecen en la copia.
- **No sustituye al backup** — es para probar, no para recuperar ante un desastre.

## Buenas prácticas avanzadas

- **Nunca compartas el filestore entre dos bases de datos** — dale a la copia su propia carpeta, no un enlace a la de producción. Odoo limpia automáticamente los adjuntos huérfanos: si dos bases apuntan al mismo directorio, la limpieza de una puede borrar ficheros que la otra todavía usa... incluida producción.
- **La copia conserva la identidad de producción** — parámetros como `web.base.url` y el UUID de la base de datos viajan con ella: los enlaces que genere apuntarán al dominio real y, en la edición Enterprise, dos bases con el mismo UUID hacen que Odoo detecte un duplicado y muestre avisos de expiración. Revísalos (*Ajustes técnicos ▸ Parámetros del sistema*) después de duplicar.
- **Cronometra la duplicación sobre tus datos reales** — con una base grande, `pg_restore` más la copia del filestore pueden tardar más de lo que crees. Mídelo una vez para saber cuánta ventana necesitas y no quedarte a medias en plena jornada.

---

*En resumen: duplicar te da una réplica realista de producción para poder probar; pero es solo la primera mitad —una copia sin [neutralizar](Neutralizar-Base-de-Datos.md) sigue siendo tan peligrosa como el original.*
