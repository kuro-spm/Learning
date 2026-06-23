# NAS

## ¿Qué es?

Un **NAS** (*Network Attached Storage*, "almacenamiento conectado a la red") es un pequeño aparato con uno o varios discos duros que se conecta a la red para guardar archivos y servirlos a todos los equipos. En la práctica es un **disco duro compartido para toda la red**, encendido siempre y accesible desde cualquier dispositivo autorizado.

## ¿Por qué existe?

Guardar archivos importantes en el disco de un portátil es arriesgado: si se rompe o se pierde, los archivos se van con él. Y compartir una carpeta desde el ordenador de alguien obliga a que ese equipo esté siempre encendido. El NAS resuelve ambas cosas: es un dispositivo dedicado solo a almacenar y compartir, diseñado para estar disponible 24/7 y, normalmente, para proteger los datos aunque falle un disco.

> Piensa en el NAS como "tu propia nube en casa o en la oficina": el almacenamiento de un servicio como Dropbox, pero el aparato es tuyo y está en tu red.

## ¿Cuándo y para qué se usa?

- **Almacén central de archivos** de una oficina o una familia: todos guardan ahí en lugar de en discos sueltos.
- **Copias de seguridad** automáticas de varios equipos en un mismo sitio.
- **Servidor multimedia**: guardar y reproducir fotos, vídeos o música en el televisor y los móviles.
- **Acceso remoto a tus archivos** desde fuera, a través de internet de forma controlada.

## Lo mínimo que necesitas saber

**1. Se administra desde el navegador**

Casi todos los NAS tienen un panel web. Te conectas a su dirección y gestionas usuarios, carpetas y permisos desde ahí:

```text
http://192.168.1.60
```

**2. Comparte por los mismos protocolos de siempre**

Un NAS no inventa nada nuevo: ofrece sus carpetas usando [SMB](Carpeta-Compartida.md), FTP, etc. Por eso accedes a él igual que a cualquier carpeta compartida:

```text
\\192.168.1.60\fotos
```

**3. Varios discos para no perder datos (RAID)**

Muchos NAS usan **RAID**: combinan varios discos de modo que, si uno falla, los datos siguen estando en los otros. Es protección frente a averías, **no** una copia de seguridad por sí sola.

**4. Usuarios y permisos**

Como un servidor, distingue usuarios: cada persona entra con su cuenta y ve solo las carpetas a las que tiene acceso.

## Lo que NO hace

- **RAID no sustituye a las copias de seguridad** — protege frente a un disco roto, pero no frente a un borrado por error, un virus o un incendio. Sigues necesitando copias aparte.
- **No es magia infinita** — tiene una capacidad limitada por sus discos; cuando se llena, hay que ampliar o liberar espacio.
- **No es seguro abrirlo a internet sin cuidado** — exponer un NAS al exterior sin protección es un riesgo; el acceso remoto debe configurarse con cabeza (contraseñas fuertes, [VPN](VPN.md)...).

---

*En resumen: un NAS es un disco duro dedicado y siempre encendido que vive en la red para que todos tus equipos guarden y compartan archivos en un único sitio seguro.*
