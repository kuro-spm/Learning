# FTP y SFTP

**¿Qué es?** **FTP** (*File Transfer Protocol*) es un protocolo veterano pensado específicamente para **transferir archivos** entre tu equipo y un servidor a través de la red: subirlos y descargarlos. **SFTP** es su versión segura, que viaja cifrada por dentro de [SSH](SSH.md).

---

## ¿Por qué existe?

Antes de la web tal y como la conocemos, ya hacía falta una forma estándar de mover ficheros entre máquinas: enviar los archivos de una página al servidor que la aloja, descargar un programa, intercambiar documentos grandes. FTP nació para eso y sigue muy presente.

El problema es que el FTP clásico envía todo (incluida la contraseña) sin cifrar. **SFTP** soluciona ese fallo aprovechando el canal seguro de SSH, por lo que hoy es la opción recomendada.

> Si una carpeta compartida es "abrir un cajón común", FTP/SFTP es más bien "una ventanilla de envíos": no trabajas dentro del archivo remoto, lo subes o lo bajas.

---

## ¿Cuándo y para qué se usa?

- **Publicar una página web**: subir los archivos del sitio al servidor que lo aloja.
- **Intercambiar archivos con un servidor** cuando no quieres montarlo como carpeta de red.
- **Descargas masivas** de un servidor de archivos.

---

## Lo mínimo que necesitas saber

**1. Te conectas con un cliente FTP**

Programas como FileZilla, o el propio terminal, piden servidor, usuario y contraseña:

```text
Servidor:  ftp.miservidor.com
Usuario:   juan
Puerto:    21   (FTP)   /   22   (SFTP, va por SSH)
```

**2. Subir y bajar por terminal (SFTP)**

```bash
sftp usuario@192.168.1.50
put informe.pdf          # subir un archivo al servidor
get backup.zip           # descargar un archivo del servidor
```

**3. Prefiere SFTP siempre que puedas**

Mismo manejo que FTP, pero cifrado. Salvo que un sistema antiguo te obligue a usar FTP a secas, usa SFTP.

---

## Lo que NO hace

- **FTP clásico no cifra** — usuario y contraseña viajan en texto plano; por eso existe SFTP.
- **No es para trabajar "en vivo" sobre el archivo** — transfieres copias; para editar directamente en remoto encaja mejor una [carpeta compartida](Carpeta-Compartida.md).
- **No sincroniza solo** — sube y baja cuando tú lo pides; no mantiene dos carpetas iguales automáticamente.

---

*En resumen: FTP/SFTP es la ventanilla de subida y bajada de archivos contra un servidor — y entre los dos, SFTP es el que va cifrado y deberías preferir.*
