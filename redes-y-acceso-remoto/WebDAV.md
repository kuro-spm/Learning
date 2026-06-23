# WebDAV

**¿Qué es?** **WebDAV** (*Web Distributed Authoring and Versioning*) es una extensión del protocolo de la web (HTTP, el mismo que usa tu navegador) que permite **leer y guardar archivos en un servidor remoto como si fuera una carpeta de tu equipo**, pero a través de internet.

---

## ¿Por qué existe?

Las [carpetas compartidas (SMB)](Carpeta-Compartida.md) son perfectas para la red local, pero no están pensadas para funcionar bien a través de internet. Por otro lado, [FTP](FTP-y-SFTP.md) sirve para transferir archivos, pero no para trabajar con ellos de forma cómoda como si estuvieran en tu disco.

WebDAV combina lo mejor de ambos: usa HTTP (que atraviesa internet sin problemas y es fácil de cifrar) y permite montar una carpeta remota directamente en el explorador de archivos.

> Si SMB es "la carpeta compartida de la oficina" y FTP "la ventanilla de envíos", WebDAV es "una carpeta de la nube que aparece como una unidad más de tu equipo".

---

## ¿Cuándo y para qué se usa?

- **Servicios de nube** que ofrecen acceso a tus archivos como una carpeta (Nextcloud, ownCloud y muchos [NAS](NAS.md) lo soportan).
- **Editar documentos en remoto** directamente, sin descargar y volver a subir.
- **Acceder a archivos a través de internet** cuando SMB no es viable.

---

## Lo mínimo que necesitas saber

**1. Se conecta con una dirección web (URL)**

A diferencia de SMB (`\\equipo\carpeta`), WebDAV usa una URL normal:

```text
https://nube.miservidor.com/remote.php/dav/files/juan/
```

**2. Se monta como unidad de red**

Tanto Windows como Linux y macOS pueden "mapear" esa URL como si fuera un disco más, y luego trabajas con los archivos con normalidad.

**3. Usa siempre HTTPS**

WebDAV puede ir por HTTP (sin cifrar) o por **HTTPS** (cifrado). Igual que en cualquier web, usa siempre la versión segura para que tus credenciales y archivos no viajen al descubierto.

---

## Lo que NO hace

- **No es tan rápido como SMB en red local** — al ir sobre HTTP, suele rendir peor para muchos archivos pequeños.
- **No sincroniza solo** — da acceso al archivo remoto, pero no mantiene una copia local actualizada como hacen las apps de escritorio de la nube.
- **No controla versiones por sí mismo** en la práctica — aunque la "V" de su nombre lo prometía, el versionado lo aporta normalmente el servicio que hay detrás.

---

*En resumen: WebDAV es la carpeta compartida que viaja por la web — monta archivos remotos como una unidad de tu equipo usando el mismo protocolo (y la misma seguridad) que tu navegador.*
