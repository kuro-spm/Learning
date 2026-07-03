# Red local e IP

**¿Qué es?** Una **red local** es el conjunto de equipos conectados entre sí en un mismo sitio (una oficina, una casa) que pueden hablarse directamente. Una **dirección IP** es el "número de teléfono" de cada equipo dentro de esa red: lo identifica para que los demás sepan a dónde enviar los datos.

---

## ¿Por qué existe?

Para que dos equipos se comuniquen, cada uno necesita una forma de localizar al otro. Igual que para enviar una carta necesitas una dirección postal, para enviar datos por la red necesitas una dirección IP.

Casi todas las conexiones de esta guía (escritorio remoto, carpetas compartidas, SSH...) se reducen a lo mismo: "conéctate al equipo que está en **esta IP**, por **este puerto**". Si entiendes IP y puerto, el resto encaja solo.

> Piensa en la IP como la dirección de un edificio y en el puerto como el número de apartamento. La IP te lleva al equipo correcto; el puerto, al servicio correcto dentro de ese equipo.

---

## ¿Cuándo y para qué se usa?

Aparece siempre que conectas dos máquinas: acceder al ordenador de la oficina desde casa, entrar a un servidor, abrir una carpeta compartida o configurar una impresora de red. En todos esos casos lo primero es saber **qué IP** tiene el equipo destino.

---

## Lo mínimo que necesitas saber

**1. Ver tu propia IP**

```bash
# Windows
ipconfig

# Linux / macOS
ip addr        # o el clásico: ifconfig
```

**2. IP privada vs. IP pública**

- **IP privada**: la que tienes dentro de tu red local. Suele empezar por `192.168.x.x` o `10.x.x.x`. Solo es visible para los equipos de tu misma red.
- **IP pública**: la que te ve internet (te la da tu proveedor). Es la dirección del router de toda tu red de cara al exterior.

**3. El puerto: el servicio concreto**

Un mismo equipo ofrece varios servicios a la vez. El puerto dice a cuál quieres llegar. Algunos habituales:

```text
22    → SSH
3389  → Escritorio remoto (RDP)
445   → Carpetas compartidas (SMB)
21    → FTP
```

**4. Probar si llegas a un equipo**

```bash
ping 192.168.1.50          # ¿responde el equipo?
```

**5. Nombre en lugar de número**

A veces puedes usar el **nombre** del equipo en vez de su IP (`miservidor`, `nas-oficina`). Es más fácil de recordar; por detrás se traduce al número igualmente.

---

## Lo que NO hace

- **No protege la conexión** — saber la IP no cifra nada; de la seguridad se encargan SSH, VPN, etc.
- **No garantiza acceso** — puedes "ver" un equipo y aun así necesitar usuario y contraseña.
- **No es fija por defecto** — muchas IPs privadas cambian con el tiempo salvo que se reserven a propósito.

---

## Buenas prácticas avanzadas

- **Que el ping falle no significa que el equipo esté caído** — muchos cortafuegos bloquean el ping pero dejan pasar los servicios. La prueba fiable es contra el puerto concreto: `Test-NetConnection 192.168.1.50 -Port 3389` en Windows, o `nc -zv 192.168.1.50 22` en Linux.
- **Reserva las IPs en el router, no en el equipo** — para que un equipo tenga siempre la misma IP, mejor que configurarla a mano en el propio equipo es crear una *reserva DHCP* en el router (asociando su dirección MAC a una IP concreta). Así toda la asignación vive en un único sitio y evitas el clásico conflicto de dos equipos peleándose por la misma dirección.

---

*En resumen: una IP es la dirección de un equipo en la red y el puerto el servicio concreto dentro de él; toda conexión empieza por saber a qué IP y a qué puerto vas.*
