# Redirección de Puertos (Port Forwarding)

## ¿Qué es?

La **redirección de puertos** es una regla que se configura en el router para que las conexiones que llegan desde internet a un puerto concreto se reenvíen automáticamente a un equipo determinado de tu red local. Es la forma de hacer que un servicio interno (un servidor, una cámara, un [NAS](NAS.md)) sea accesible **desde fuera**.

## ¿Por qué existe?

Los equipos de tu red local tienen IPs privadas que internet no ve: de cara al exterior, todos comparten la única IP pública del router. Por eso, cuando alguien intenta conectar desde fuera, el router no sabe a cuál de tus equipos debe enviar esa petición.

La redirección de puertos resuelve esa duda con una instrucción clara: *"todo lo que llegue al puerto X, mándalo al equipo Y"*.

> Imagina un edificio con una sola dirección postal (la IP pública) y muchos apartamentos (tus equipos). El conserje (el router) necesita una nota que diga "los paquetes del buzón 3389 son para el apartamento 192.168.1.50". Esa nota es la redirección de puertos.

## ¿Cuándo y para qué se usa?

- **Publicar un servicio propio**: que un servidor web o de juegos casero sea visible en internet.
- **Acceder a casa desde fuera**: llegar a tu NAS o a una cámara de vigilancia cuando no estás.
- **Conexiones que no pasan por un intermediario**, a diferencia de [herramientas como TeamViewer](Herramientas-de-Control-Remoto.md).

## Lo mínimo que necesitas saber

**1. La regla se crea en el router**

Entras en el panel del router y defines: puerto de entrada, equipo destino (su IP local) y puerto destino.

```text
Puerto externo:  3389
IP interna:      192.168.1.50
Puerto interno:  3389
Protocolo:       TCP
```

**2. El equipo destino debería tener IP fija**

Si la IP local del equipo cambia, la regla apunta al sitio equivocado. Por eso conviene reservarle una IP fija en el router.

**3. Te conectas usando la IP pública**

Desde fuera ya no usas la IP local, sino la pública del router (o un nombre de dominio asociado):

```text
mstsc /v:203.0.113.10        (IP pública del router, no la 192.168.x.x)
```

**4. Abrir un puerto es abrir una puerta**

Cualquiera en internet podrá intentar conectar a ese puerto. Solo redirige servicios bien protegidos y, si puedes, cambia el puerto por defecto para no ser un blanco obvio.

## Lo que NO hace

- **No añade seguridad** — solo abre el acceso; la protección depende del servicio que hay detrás. Exponer escritorio remoto o SMB así es arriesgado.
- **No cifra nada** — a diferencia de una [VPN](VPN.md), no protege la conexión; en muchos casos la VPN es la alternativa recomendada.
- **No funciona si tu proveedor no te da IP pública propia** — algunas conexiones comparten IP (CGNAT) y la redirección no surte efecto.

---

*En resumen: la redirección de puertos es la nota que le dejas al router para que reparta las conexiones de internet al equipo correcto de tu red — potente, pero hay que usarla con cabeza porque abre una puerta al exterior.*
