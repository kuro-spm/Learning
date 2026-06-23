# VPN

## ¿Qué es?

Una **VPN** (*Virtual Private Network*, "red privada virtual") crea un **túnel cifrado** entre tu equipo y una red remota, de modo que es como si estuvieras físicamente enchufado a esa red aunque estés a kilómetros. Tu equipo pasa a comportarse como uno más de la red de destino.

## ¿Por qué existe?

Muchos servicios (carpetas compartidas, escritorio remoto, un NAS...) están pensados para usarse **solo dentro de la red local** y es peligroso abrirlos a internet. Pero a veces necesitas usarlos desde fuera: desde casa, de viaje, desde otra oficina.

La VPN resuelve esto sin exponer nada: en lugar de abrir cada servicio al exterior, abres **una sola puerta segura** (la VPN) y, una vez dentro, accedes a todo como si estuvieras en la oficina. Además, todo lo que pasa por el túnel va cifrado, incluso si estás en una wifi pública.

> Imagina un túnel privado y blindado que conecta tu portátil directamente con la oficina, atravesando internet sin que nadie pueda ver lo que circula por dentro.

## ¿Cuándo y para qué se usa?

- **Teletrabajo seguro**: acceder a las carpetas compartidas, servidores y recursos internos de la oficina desde casa.
- **Proteger la conexión en redes públicas**: cifrar tu tráfico en la wifi de un hotel o una cafetería.
- **Unir sedes**: conectar dos oficinas como si fueran una sola red.
- **Servir de antesala** para otros accesos: primero entras por VPN y luego usas [escritorio remoto](Escritorio-Remoto.md) o una [carpeta compartida](Carpeta-Compartida.md) con seguridad.

## Lo mínimo que necesitas saber

**1. Hace falta un servidor VPN y un cliente**

En la red de destino hay un **servidor VPN** que acepta conexiones; en tu equipo, una **app cliente** que las inicia (OpenVPN, WireGuard, el cliente VPN de Windows...).

**2. Te conectas con una configuración o credenciales**

El administrador te entrega un archivo de configuración o unas credenciales. Al conectar, tu equipo recibe una IP de la red remota:

```text
Conectado a "VPN-Oficina"
Tu IP en la red de la oficina: 10.8.0.4
```

**3. Una vez dentro, usas la red como si estuvieras allí**

Ya puedes acceder a recursos internos por su dirección local, igual que si estuvieras en la oficina:

```bash
ping 192.168.1.50          # un equipo interno, ahora alcanzable
\\192.168.1.50\documentos   # su carpeta compartida
```

**4. "Túnel completo" vs. "túnel dividido"**

Puedes configurar que **todo** tu tráfico de internet pase por la VPN (más privacidad) o que solo pase el dirigido a la red remota y el resto salga normal (más rápido). Cada opción tiene su sentido.

## Lo que NO hace

- **No te hace anónimo del todo** — una VPN cifra y redirige tu tráfico, pero quien gestiona el servidor VPN sí puede verlo. No es una capa mágica de anonimato.
- **No sustituye a las contraseñas** — abre la puerta a la red, pero cada servicio interno sigue pidiendo su propio usuario y contraseña.
- **No acelera tu conexión** — al cifrar y dar un rodeo, puede incluso ralentizarla un poco; su objetivo es la seguridad y el acceso, no la velocidad.

---

*En resumen: una VPN te mete dentro de una red remota a través de un túnel cifrado, para usar sus recursos con seguridad como si estuvieras físicamente allí.*
