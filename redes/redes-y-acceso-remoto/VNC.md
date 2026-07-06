# VNC

**¿Qué es?** **VNC** (*Virtual Network Computing*) es una tecnología para ver y controlar el escritorio de otro equipo a través de la red, igual que el escritorio remoto, pero **multiplataforma** y basada en un estándar abierto: funciona entre Windows, Linux y macOS indistintamente.

---

## ¿Por qué existe?

El Escritorio Remoto de Windows ([RDP](Escritorio-Remoto.md)) es cómodo, pero su servidor solo vive en Windows. VNC nació para resolver lo mismo sin atarse a un sistema operativo: controlar un Linux desde un Windows, un Mac desde un Linux, etc. Comparte la pantalla "tal cual", enviando imágenes de lo que se ve y devolviendo tus clics y pulsaciones.

> Si RDP es la solución "de Windows para Windows", VNC es la solución "para todos con todos".

---

## ¿Cuándo y para qué se usa?

- Controlar equipos **Linux o macOS** en remoto, donde RDP no encaja bien.
- **Ver exactamente la misma pantalla** que tiene delante otra persona (a diferencia de RDP, VNC suele mostrar la sesión activa, no abrir una nueva): ideal para soporte y formación.
- Acceder a dispositivos pequeños (como un mini-PC o una placa tipo Raspberry Pi) que no tienen monitor propio.

---

## Lo mínimo que necesitas saber

**1. Hace falta un servidor y un visor**

En el equipo a controlar instalas un **servidor VNC**; en el tuyo, un **visor VNC**. Hay muchas implementaciones (TightVNC, RealVNC, TigerVNC...), compatibles entre sí en lo básico.

**2. Te conectas por IP y puerto**

El puerto típico es el **5900**. En el visor indicas:

```text
192.168.1.50:5900
```

**3. Ciframos por separado**

VNC por sí solo manda la imagen sin cifrar fuerte. Lo habitual es protegerlo metiéndolo dentro de un túnel [SSH](SSH.md) o de una [VPN](VPN.md).

---

## Lo que NO hace

- **No cifra de forma robusta por defecto** — no lo expongas a internet "a pelo".
- **No transfiere archivos por sí mismo** — solo comparte pantalla y control.
- **No es tan fluido como RDP** en redes lentas, porque envía imágenes de la pantalla en lugar de instrucciones de dibujo.

---

## Buenas prácticas avanzadas

- **Ata el servidor VNC a `localhost` y entra por túnel SSH** — la configuración experta no expone el 5900 ni siquiera en la red local: el servidor solo escucha en `localhost` y el túnel se crea al conectar, `ssh -L 5900:localhost:5900 usuario@equipo`; en el visor abres `localhost:5900`. Conexión cifrada y ningún puerto VNC a la vista.
- **La contraseña "de VNC" clásica no es una contraseña de verdad** — el protocolo tradicional la trunca a 8 caracteres y la protege con un cifrado (DES) trivial de romper. Trátala como un simple pestillo y pon la seguridad real en el túnel SSH/VPN o en una implementación moderna con autenticación propia.
- **Los puertos van por "displays"** — en Linux, cada sesión VNC es un *display*: `:1` escucha en el puerto 5901, `:2` en el 5902... Cuando el visor no conecta "al 5900", casi siempre es que el servidor está sirviendo el display `:1` y hay que apuntar al 5901.

---

*En resumen: VNC es el "escritorio remoto universal" que conecta cualquier sistema operativo con cualquier otro compartiendo la pantalla real del equipo.*
