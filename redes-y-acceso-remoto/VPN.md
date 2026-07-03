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

## Buenas prácticas avanzadas

- **Evita que las redes se solapen** — el fallo clásico: tu casa usa `192.168.1.x` y la oficina también, así que al conectar la VPN tu equipo no sabe si `192.168.1.50` está a tu lado o al otro lado del túnel, y las conexiones fallan de forma intermitente. Quien monta un servidor VPN elige a propósito un rango poco común para la red interna (p. ej. `10.83.24.0/24`), precisamente para no chocar con las redes domésticas típicas.
- **El DNS forma parte del túnel** — si conectas la VPN pero sigues usando el DNS de tu casa, los nombres internos (`nas-oficina`, `intranet`) no resolverán aunque el túnel funcione y las IPs respondan. La configuración de la VPN debe empujar también el servidor DNS interno; cuando "la VPN va pero no encuentro nada por nombre", este es el sospechoso número uno.
- **La VPN te mete en la red, pero no debería darte toda la red** — un portátil doméstico comprometido conectado a una VPN con acceso total es una autopista hacia dentro. En una configuración cuidada, cada perfil de VPN solo alcanza las IPs y puertos que necesita: quien administra los servidores llega a los servidores, no a las carpetas de contabilidad.
- **WireGuard es la opción moderna por algo más que la velocidad** — su servidor no responde nada en absoluto a paquetes no autenticados, así que un escáner de internet ni siquiera detecta que hay una VPN escuchando; y su configuración cabe en veinte líneas, lo que deja mucho menos margen de error que un OpenVPN mal afinado.

---

*En resumen: una VPN te mete dentro de una red remota a través de un túnel cifrado, para usar sus recursos con seguridad como si estuvieras físicamente allí.*
