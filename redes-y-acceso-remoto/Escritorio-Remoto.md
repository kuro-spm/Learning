# Escritorio Remoto (RDP)

## ¿Qué es?

El **Escritorio Remoto** te permite ver y controlar la pantalla de otro ordenador desde el tuyo, como si estuvieras sentado delante de él: mismo escritorio, mismos programas, mismo ratón y teclado. En Windows, la tecnología que lo hace posible se llama **RDP** (*Remote Desktop Protocol*).

## ¿Por qué existe?

Muchas veces necesitas usar un equipo que no tienes físicamente delante: el ordenador de la oficina cuando trabajas desde casa, un servidor sin pantalla ni teclado, o el equipo de un compañero al que ayudas. En lugar de desplazarte, el escritorio remoto "te teletransporta" delante de esa máquina a través de la red.

> Si has usado alguna vez TeamViewer o AnyDesk para que alguien controle tu pantalla, RDP es la versión integrada en Windows de esa misma idea.

## ¿Cuándo y para qué se usa?

- **Teletrabajo**: usar tu equipo de la oficina desde casa con todos sus programas y archivos.
- **Administrar servidores**: muchos servidores no tienen monitor; te conectas a su escritorio para gestionarlos.
- **Soporte técnico**: entrar en el equipo de otra persona para resolver un problema.
- **Software pesado**: ejecutar un programa que solo está instalado en una máquina potente, controlándolo desde un portátil modesto.

## Lo mínimo que necesitas saber

**1. Activar el escritorio remoto en el equipo destino**

En el equipo al que quieres conectarte (Windows): *Configuración → Sistema → Escritorio remoto → Activar*. A partir de ahí ese equipo "escucha" peticiones por el puerto **3389**.

**2. Conectarte desde el cliente**

En tu equipo abres la app **Conexión a Escritorio remoto** (`mstsc`) e indicas la dirección del equipo destino:

```text
Equipo:   192.168.1.50        (la IP o el nombre del equipo destino)
Usuario:  juan
```

**3. Conexión por línea de comandos**

```bash
mstsc /v:192.168.1.50
```

**4. Solo puede usarlo una persona a la vez (en Windows de escritorio)**

Cuando te conectas, la sesión "se mueve" a tu pantalla; quien estuviera físicamente delante ve la pantalla bloqueada. No es una pantalla compartida, es un relevo.

**5. Necesitas una cuenta válida en el equipo destino**

No basta con llegar a la IP: tienes que iniciar sesión con un usuario y contraseña que existan en esa máquina.

## Lo que NO hace

- **No funciona si el equipo destino está apagado o suspendido** — tiene que estar encendido y conectado a la red.
- **No es seguro exponerlo directamente a internet** — abrir el puerto 3389 al exterior es un riesgo conocido; lo habitual es entrar antes por una [VPN](VPN.md).
- **No copia archivos por sí mismo** — controlas la pantalla, pero mover ficheros entre equipos requiere otra herramienta (o activar la opción de unidades compartidas).
- **No es exclusivo de Windows como cliente** — hay apps de RDP para macOS, Linux, móviles..., aunque el servidor RDP es propio de Windows.

---

*En resumen: el escritorio remoto te sienta virtualmente delante de otro ordenador para usarlo como si fuera el tuyo, a través de la red.*
