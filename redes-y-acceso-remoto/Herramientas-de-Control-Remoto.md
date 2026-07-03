# Herramientas de Control Remoto (TeamViewer, AnyDesk...)

**¿Qué es?** Son programas como **TeamViewer**, **AnyDesk** o **Chrome Remote Desktop** que permiten ver y controlar la pantalla de otro equipo a través de internet **sin configurar nada en la red**. Hacen lo mismo que el escritorio remoto o VNC, pero pensados para funcionar "a la primera" entre cualquier par de equipos del mundo.

---

## ¿Por qué existe?

[RDP](Escritorio-Remoto.md) y [VNC](VNC.md) funcionan muy bien dentro de la red local, pero usarlos desde fuera obliga a tocar el router, abrir puertos o montar una [VPN](VPN.md). Eso es complicado para mucha gente.

Estas herramientas resuelven el problema con una idea sencilla: ambos equipos se conectan a un **servidor intermedio** de la empresa que hace de puente. Así no hace falta configurar nada de red; basta con instalar el programa en los dos lados.

> Si RDP/VNC son "llamar directamente al número de teléfono de un equipo", estas herramientas son "una centralita que pone en contacto a los dos sin que ninguno conozca el número del otro".

---

## ¿Cuándo y para qué se usa?

- **Soporte técnico a distancia**: ayudar a un familiar o a un cliente entrando en su pantalla, esté donde esté.
- **Acceder a tu propio equipo** desde fuera sin pelearte con la configuración del router.
- **Demostraciones y formación** compartiendo lo que ves en tiempo real.

---

## Lo mínimo que necesitas saber

**1. Se instala en ambos equipos**

El que controla y el controlado usan el mismo programa. El equipo controlado muestra un **ID** y una **contraseña** temporal.

```text
Tu ID:          123 456 789
Contraseña:     a1b2c3
```

**2. Conectas con el ID, no con la IP**

No necesitas saber la dirección de red del otro equipo: introduces su ID y la contraseña, y la herramienta hace el resto por internet.

**3. Suele incluir transferencia de archivos y chat**

A diferencia de RDP/VNC "a pelo", estas apps añaden extras: pasar ficheros, chatear o reiniciar el equipo manteniendo la sesión.

**4. Ojo con el uso comercial**

Muchas son gratis para uso personal pero de pago para empresas. Conviene revisar la licencia.

---

## Lo que NO hace

- **No funciona sin internet** — dependen del servidor intermedio de la empresa; si ese servicio cae, no hay conexión.
- **No es la opción más privada** — tu sesión pasa por servidores de un tercero. Para entornos sensibles suele preferirse VPN + RDP.
- **No sustituye a una buena contraseña** — son un objetivo habitual de estafas ("instala esto y dame tu ID"); nunca des acceso a alguien en quien no confíes.

---

## Buenas prácticas avanzadas

- **Tu cuenta del servicio es la llave maestra: protégela con 2FA** — si configuras *acceso desatendido* a tus equipos, quien entre en tu cuenta de TeamViewer o AnyDesk entra en todos ellos a la vez. Activa la verificación en dos pasos en la cuenta y usa una contraseña de acceso desatendido distinta por equipo, nunca la misma reciclada.
- **Activa la lista blanca de IDs** — casi todas estas herramientas permiten restringir las conexiones entrantes a una lista de IDs o cuentas concretas. Con eso, la contraseña deja de ser la única barrera: aunque alguien la consiga, su ID no está autorizado. Y échale un ojo de vez en cuando al registro de conexiones entrantes que guarda la propia aplicación: ahí se ve quién ha entrado y cuándo.

---

*En resumen: TeamViewer, AnyDesk y similares son el control remoto "sin configuración", que conecta dos equipos por internet a través de un servidor puente a cambio de algo de privacidad y dependencia de un tercero.*
