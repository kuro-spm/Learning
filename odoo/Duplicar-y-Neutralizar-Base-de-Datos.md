# Duplicar y neutralizar la base de datos

## ¿Qué es?

Es la técnica central para probar con seguridad: hacer una **copia** de la base de datos de producción y **neutralizarla**, es decir, desactivar todo lo que podría tener un efecto en el mundo real (enviar correos, cobrar pagos, sincronizar con sistemas externos). El resultado es un gemelo con datos realistas donde puedes hacer lo que quieras sin que nadie se entere.

## ¿Por qué existe?

Probar sobre datos inventados no siempre revela los problemas: el error puede aparecer solo con la configuración y el volumen reales. Por eso se prueba sobre una **copia de los datos de verdad**.

Pero una copia sin más es peligrosa: si arranca con las mismas conexiones que producción, podría enviar correos reales, ejecutar cobros reales o llamar a servicios externos reales. Neutralizar corta todos esos hilos con el exterior. Así tienes realismo sin riesgo.

> Piensa en un simulador de vuelo: reproduce fielmente el avión de verdad, pero por mucho que estrelles el simulador, ningún avión real se cae.

## ¿Cuándo y para qué se usa?

- Antes de **instalar o actualizar un módulo**, para ver el efecto sobre datos reales.
- Para **ensayar una migración** a una versión nueva de Odoo.
- Para **reproducir un error** que solo ocurre con los datos de producción.
- Para **formar** o hacer **demos** con información realista pero sin consecuencias.

## Lo mínimo que necesitas saber

**1. Duplicar la base de datos**

En el gestor de bases de datos de Odoo (`/web/database/manager`) hay un botón **Duplicate**. También se puede por línea de comandos, restaurando un backup con otro nombre:

```bash
# Restaurar una copia de producción con un nombre distinto (una copia de pruebas)
createdb odoo_pruebas
pg_restore -d odoo_pruebas produccion.dump
# y copiar también el filestore (adjuntos, imágenes) a la carpeta de la nueva BD
```

**2. Neutralizar: el paso que no puedes olvidar**

Odoo trae un comando específico que desactiva todo lo peligroso de golpe (servidores de correo, tareas programadas, conexiones de pago, etc.):

```bash
# Arrancar Odoo sobre la copia y neutralizarla
odoo-bin neutralize -d odoo_pruebas
```

Desde Odoo 16 esto está integrado; en Odoo.sh y Odoo.com la neutralización se aplica **automáticamente** al crear un entorno de staging.

**3. Qué apaga la neutralización (lo esencial)**

- **Correo saliente** — para no enviar mensajes reales a clientes reales.
- **Tareas programadas (*cron*)** — para que no se disparen procesos automáticos (facturación recurrente, recordatorios...).
- **Pasarelas de pago y bancos** — se ponen en modo prueba o se desconectan.
- **Integraciones externas** — claves de API y sincronizaciones se desactivan.

**4. Comprueba que de verdad está neutralizada**

Antes de tocar nada, verifica que el correo saliente no apunta a un servidor real y que los *crons* peligrosos están desactivados. Una copia mal neutralizada es casi tan arriesgada como probar en producción.

```python
# Señal típica: los servidores de correo saliente quedan desactivados
# Ajustes técnicos ▸ Correo electrónico ▸ Servidores de correo saliente → sin activos
```

## Lo que NO hace

- **No es automático si lo montas tú** — en una instalación propia, duplicar no neutraliza; el paso `neutralize` es tuyo.
- **No actualiza sola la copia** — es una foto del momento; los datos nuevos de producción no aparecen ahí.
- **No sustituye al backup** — es para probar, no para recuperar; y una copia no neutralizada no debe convivir con producción.

---

*En resumen: duplicar te da datos realistas para probar y neutralizar corta todos los hilos con el mundo real; juntas, convierten una copia de producción en un simulador seguro donde nada de lo que hagas escapa al exterior.*
