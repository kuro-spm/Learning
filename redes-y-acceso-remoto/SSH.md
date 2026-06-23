# SSH

## ¿Qué es?

**SSH** (*Secure Shell*) es una forma segura de conectarte a otro equipo a través de la red para darle órdenes por **línea de comandos**. Todo lo que viaja entre los dos equipos va **cifrado**, así que nadie que espíe la red puede leer tus comandos ni tus contraseñas.

## ¿Por qué existe?

Antes existían herramientas (como Telnet) que permitían entrar en un equipo remoto, pero enviaban todo —incluida la contraseña— en texto plano: cualquiera que interceptara la red lo veía. SSH nació para hacer lo mismo, pero con la comunicación cifrada de extremo a extremo.

A diferencia del escritorio remoto, SSH no te muestra una pantalla con ventanas: te da una **consola de texto**. Es más ligero y rapidísimo, ideal para servidores que ni siquiera tienen interfaz gráfica.

> Si el escritorio remoto es "ver y tocar la pantalla del otro equipo", SSH es "escribirle órdenes por una terminal segura".

## ¿Cuándo y para qué se usa?

- **Administrar servidores**: la mayoría de servidores Linux se gestionan exclusivamente por SSH.
- **Desplegar aplicaciones**: conectarte a un servidor para actualizar el código, reiniciar un servicio o revisar logs.
- **Ejecutar comandos en remoto** sin la pesadez de cargar un escritorio entero.
- **Servir de "túnel seguro"** para otras conexiones (por ejemplo, proteger una sesión [VNC](VNC.md) metiéndola dentro de SSH).

## Lo mínimo que necesitas saber

**1. Conectarte a un equipo**

```bash
ssh usuario@192.168.1.50
# La primera vez te pedirá confirmar la identidad del servidor y luego la contraseña
```

**2. Indicar un puerto distinto**

Por defecto SSH usa el puerto **22**. Si el servidor usa otro:

```bash
ssh usuario@192.168.1.50 -p 2222
```

**3. Claves en vez de contraseña (más seguro y cómodo)**

Generas un par de claves: una **privada** (se queda en tu equipo, secreta) y una **pública** (la copias al servidor). Si encajan, entras sin escribir contraseña.

```bash
ssh-keygen                          # crea el par de claves
ssh-copy-id usuario@192.168.1.50    # instala tu clave pública en el servidor
```

> Es como un candado (clave pública, en el servidor) y su única llave (clave privada, en tu equipo).

**4. Copiar archivos por SSH**

```bash
scp informe.pdf usuario@192.168.1.50:/home/usuario/   # subir un archivo
scp usuario@192.168.1.50:/var/log/app.log .            # descargarlo
```

**5. Ejecutar un comando sin abrir sesión interactiva**

```bash
ssh usuario@192.168.1.50 "df -h"    # ejecuta el comando y devuelve el resultado
```

## Lo que NO hace

- **No te da una interfaz gráfica** — trabajas con texto (aunque existen extensiones para reenviar ventanas).
- **No funciona si el servidor no tiene SSH activado** — el equipo destino debe ejecutar un servicio SSH escuchando.
- **No protege de contraseñas débiles** — el canal es seguro, pero una contraseña fácil sigue siendo un agujero; por eso se recomiendan las claves.

---

*En resumen: SSH es la puerta segura por la que entras a la terminal de otro equipo para darle órdenes sin que nadie pueda espiar la conexión.*
