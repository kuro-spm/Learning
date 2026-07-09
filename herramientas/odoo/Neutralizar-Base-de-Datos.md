# Neutralizar la base de datos

## ¿Qué es?

Neutralizar es **apagar en una copia de la base de datos todo lo que puede tener efecto en el mundo real**: enviar correos, ejecutar cobros, lanzar tareas programadas, sincronizar con sistemas externos. La copia sigue funcionando por dentro con datos reales, pero deja de poder comunicarse hacia fuera.

## ¿Por qué existe?

Cuando duplicas producción, la copia **hereda su configuración real**: los mismos servidores de correo, las mismas tareas programadas (*crons*), las mismas pasarelas de pago, la mensajería con las agencias de transporte (DHL, GLS, SEUR…), la conexión con Hacienda (SII/AEAT), etc. Si arrancas esa copia tal cual, **actuará como si fuera producción usando datos reales**: podría mandar correos a clientes y empleados de verdad, ejecutar cobros, generar etiquetas de envío o enviar facturas a Hacienda… **desde el entorno de pruebas.**

Neutralizar corta todos esos hilos con el exterior de golpe. Por eso se hace **antes de arrancar el servicio sobre la copia**: así el entorno de pruebas nace "mudo" hacia fuera y no existe ni un minuto de ventana en el que algo pueda escaparse.

> Piensa en el **modo avión** del móvil: el teléfono sigue encendido y puedes usar todas tus apps, pero no transmite nada hacia fuera. Una base de datos neutralizada es exactamente eso: viva por dentro, incomunicada hacia el exterior.

## ¿Cuándo y para qué se usa?

- Siempre que montes un entorno de **pruebas o *staging*** a partir de una copia de producción.
- Justo **después de duplicar** ([ver Duplicar la base de datos](Duplicar-Base-de-Datos.md)) y **antes** de arrancar Odoo sobre esa copia.
- Al **restaurar un backup** de producción en cualquier sitio que no sea producción, para poder abrirlo sin riesgo.
- Antes de **formar, hacer demos o dejar que alguien "trastee"** con datos realistas.

## Lo mínimo que necesitas saber

**1. El comando `neutralize`**

Desde Odoo 16, la neutralización viene integrada como un comando que desactiva de una vez todo lo peligroso:

```bash
# Neutralizar la copia SIN llegar a servir peticiones
odoo-bin neutralize -d odoo_test
```

En **Odoo.sh** y **Odoo Online** los entornos de *staging* se neutralizan **automáticamente** al crearlos; el gestor de bases de datos también ofrece una casilla *neutralize* al restaurar un backup. En una instalación propia (*on-premise*), el paso es **tuyo**.

**2. Hazlo antes de arrancar el servicio (no después)**

Este es el punto que no puedes saltarte. Si restauras y arrancas "solo para ver que funciona", durante esos minutos los *crons* y el correo siguen vivos y pueden disparar envíos reales. El orden correcto es siempre:

```text
restaurar la copia  →  neutralize  →  recién entonces arrancar
```

**3. Qué apaga la neutralización**

- **Correo saliente y entrante** — desactiva los servidores SMTP/IMAP para no mandar ni recibir mensajes reales.
- **Tareas programadas (*crons*)** — para que no se disparen procesos automáticos (facturación recurrente, recordatorios, sincronizaciones nocturnas).
- **Pasarelas de pago** — pasan a modo test o se desconectan; nada de cobros reales.
- **Mensajería y transportistas** (DHL, GLS, SEUR, Correos…) — no se generan envíos ni etiquetas de verdad.
- **SII/AEAT y otras conexiones fiscales** — no se envían facturas ni declaraciones a Hacienda.
- **Integraciones externas e IAP** — claves de API y servicios de Odoo (SMS, firma, etc.) se desactivan o se apuntan a un entorno de pruebas.

**4. Tus módulos a medida necesitan su propio `neutralize.sql`**

`neutralize` apaga lo que Odoo trae de serie, pero **no conoce una integración que hayas programado tú**. Desde Odoo 16, un módulo propio puede incluir un fichero `data/neutralize.sql` que el comando ejecuta junto con el resto:

```sql
-- data/neutralize.sql de tu módulo: desactivar tu sincronización a medida
UPDATE mi_integracion_config SET activo = false;
```

Si tienes integraciones custom y no las cubres así, desactívalas a mano y deja documentado cuáles son.

**5. Comprueba que de verdad quedó neutralizada**

Antes de tocar nada, verifica que no queda ningún hilo vivo hacia el exterior. Una copia mal neutralizada es casi tan arriesgada como probar en producción.

```text
Ajustes técnicos ▸ Correo electrónico ▸ Servidores de correo saliente → ninguno activo
Ajustes técnicos ▸ Automatización ▸ Acciones planificadas → los crons peligrosos, desactivados
Contabilidad/Ventas ▸ pasarelas de pago → en modo test o deshabilitadas
```

## Lo que NO hace

- **No duplica** — neutralizar actúa sobre una copia que ya existe; hacer esa copia es un paso previo aparte.
- **No es automático si lo montas tú** — en una instalación propia, restaurar un backup no neutraliza nada por sí solo.
- **No conoce tus integraciones a medida** — solo apaga lo estándar; lo que programaste tú lo tienes que cubrir con tu `neutralize.sql`.
- **No borra ni anonimiza los datos** — la copia sigue conteniendo nombres, correos y facturas reales; incomunica, pero no protege la privacidad por sí sola.

## Buenas prácticas avanzadas

- **La ventana de riesgo es el arranque, no la copia** — mientras el fichero de backup está en disco no pasa nada; el peligro empieza en el instante en que un Odoo con la config de producción se pone a servir. Neutraliza con el servicio parado y no levantes el servidor hasta haberlo hecho: así la ventana de riesgo es literalmente cero.
- **`neutralize` no anonimiza: staging es tan sensible como producción** — apagar el correo evita accidentes, pero la copia conserva datos personales reales. A efectos de RGPD, un *staging* neutralizado es tan confidencial como el original: restringe quién accede y no lo publiques en una URL abierta.
- **Cuidado con reactivar sin querer lo que neutralizaste** — restaurar en la copia un backup posterior, o reinstalar un módulo, puede volver a activar servidores de correo o *crons*. Cada vez que "refresques" el entorno de pruebas con datos nuevos, vuelve a neutralizar; no des por hecho que sigue mudo.
- **Verifica el resultado, no solo el comando** — que `neutralize` termine sin error no garantiza que tu integración concreta esté apagada (quizá te faltó el `neutralize.sql`). Antes de trabajar, envía un correo de prueba y comprueba que se queda en la cola sin salir, o revisa los servidores salientes: la confianza viene de comprobar, no de suponer.

---

*En resumen: neutralizar convierte una copia de producción en un entorno "en modo avión" —vivo por dentro, incomunicado hacia fuera— y se hace antes de arrancar el servicio para que nada de lo que pruebes se escape al mundo real.*
