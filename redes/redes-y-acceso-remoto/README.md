# Conexiones — Guía de tecnologías

Guía introductoria sobre las formas más habituales de **conectar equipos entre sí y compartir archivos** a través de la red: escritorio remoto, carpetas compartidas, NAS, SSH, VPN y compañía.

Está pensada para perfiles junior con nociones básicas de informática que se topan por primera vez con estos términos. Cada archivo explica qué es la tecnología, por qué existe, cuándo se usa y lo mínimo que necesitas saber para no perderte. No hace falta ningún conocimiento previo de redes.

---

## Orden de lectura recomendado

Sigue este orden si partes de cero: primero los cimientos, luego cómo controlar otro equipo, después cómo compartir archivos y, por último, cómo hacerlo todo de forma segura desde fuera.

### 1. Fundamentos

Antes de conectar nada, entiende cómo se localizan los equipos en la red.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Red local e IP](Red-local-e-IP.md) | La base de todo: qué es una IP y un puerto. El resto de la guía lo da por sabido. |

### 2. Controlar otro equipo

Conectarte a una máquina para usarla como si estuvieras delante de ella.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [Escritorio Remoto (RDP)](Escritorio-Remoto.md) | Ver y manejar la pantalla de otro equipo Windows. El caso más reconocible. |
| 3 | [VNC](VNC.md) | La alternativa multiplataforma al escritorio remoto. Léela tras RDP para comparar. |
| 4 | [Herramientas de Control Remoto](Herramientas-de-Control-Remoto.md) | TeamViewer, AnyDesk y similares: control remoto sin tocar la red. |
| 5 | [SSH](SSH.md) | Controlar un equipo por terminal de forma segura. Imprescindible para servidores. |

### 3. Compartir archivos

Poner ficheros en común entre varios equipos.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 6 | [Carpeta Compartida (SMB)](Carpeta-Compartida.md) | La forma clásica de compartir una carpeta en la red local. |
| 7 | [NAS](NAS.md) | Un disco compartido dedicado para toda la red. Se apoya en lo anterior. |
| 8 | [FTP y SFTP](FTP-y-SFTP.md) | Subir y bajar archivos contra un servidor. Lectura rápida. |
| 9 | [WebDAV](WebDAV.md) | Carpetas remotas a través de la web, montadas como una unidad más. |

### 4. Acceso seguro desde fuera

Cómo usar todo lo anterior desde fuera de la red local sin abrir agujeros.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 10 | [VPN](VPN.md) | La puerta segura que conecta tu equipo con una red remota. La opción recomendada para acceder desde fuera. |
| 11 | [Redirección de Puertos](Redireccion-de-Puertos.md) | La alternativa a la VPN para exponer un servicio a internet. Léela al final: hay que usarla con cuidado. |

---

> ¿Te interesa cómo se trabaja en equipo sobre el código una vez conectado? Echa un vistazo a la guía de [Git](../../devops/git/README.md).
