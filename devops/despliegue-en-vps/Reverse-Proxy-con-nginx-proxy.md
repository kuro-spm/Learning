# Reverse proxy con nginx-proxy

## ¿Qué es?

Un **reverse proxy** es el contenedor que recibe todo el tráfico web del servidor y lo reparte al contenedor que corresponde según el dominio pedido. `nginx-proxy` es una imagen que hace eso sola: vigila los contenedores que arrancan y reconfigura nginx sin que tú escribas un solo fichero de configuración.

## ¿Por qué existe?

En un servidor solo puede haber un proceso escuchando en el puerto 443. Si tienes tres aplicaciones dockerizadas, no pueden publicar las tres ese puerto: la segunda falla con `port is already allocated`.

La salida tradicional es poner nginx delante, escuchando en el 80 y el 443, y que reenvíe cada dominio al puerto interno correspondiente. Funciona, pero obliga a mantener a mano un fichero de configuración por dominio y a recargar nginx cada vez que despliegas algo nuevo. En un servidor donde las aplicaciones van y vienen, ese fichero se desincroniza en cuestión de semanas.

`nginx-proxy` invierte el planteamiento. En lugar de que tú declares las rutas en el proxy, **cada aplicación declara su propio dominio** mediante una variable de entorno, y el proxy se entera solo: está suscrito a los eventos de Docker y regenera su configuración cada vez que un contenedor arranca o para.

> Si has trabajado con inyección de dependencias, es la misma idea de inversión de control: el proxy no busca a las aplicaciones en una lista que alguien mantiene, sino que las aplicaciones se registran solas al arrancar.

## ¿Cuándo y para qué se usa?

En cualquier VPS que sirva más de una cosa por HTTP: la web pública y el panel de administración en subdominios distintos, varios clientes en el mismo servidor, o simplemente una aplicación hoy y saber que mañana habrá otra.

También en el caso de una sola aplicación, por dos motivos que compensan de sobra el contenedor extra: es la pieza que gestiona los [certificados HTTPS automáticos](HTTPS-con-Lets-Encrypt.md), y permite que la aplicación no publique ningún puerto al exterior.

---

## La pieza previa: una red compartida

El proxy y las aplicaciones son contenedores distintos, definidos en ficheros `docker-compose.yml` distintos. Para que puedan hablarse tienen que compartir una red de Docker, y esa red se crea **una sola vez, a mano**, porque no pertenece a ninguno de los dos:

```bash
docker network create proxy-net
```

Una red de Docker es una red virtual privada: los contenedores conectados a ella se ven entre sí por su nombre de contenedor, como si fueran nombres de host. `tienda-frontend` puede llamar a `http://tienda-api:3000` sin saber ninguna IP.

Verifica que existe:

```bash
docker network ls
```

```
NETWORK ID     NAME         DRIVER    SCOPE
9a1c4f8e2b13   bridge       bridge    local
4e7d2a9c1f60   proxy-net    bridge    local
```

Esta red la declararán los dos `docker-compose.yml` como `external: true`, que significa "esta red ya existe, no la crees, solo conéctate". Si te saltas este paso, el `docker compose up` falla con `network proxy-net declared as external, but could not be found`.

## El contenedor del proxy

Todo lo del proxy vive en su propia carpeta:

```bash
mkdir -p /srv/docker/nginx-proxy
cd /srv/docker/nginx-proxy
```

Y su `docker-compose.yml`:

```yaml
services:
  nginx-proxy:
    image: nginxproxy/nginx-proxy
    container_name: nginx-proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - ./certs:/etc/nginx/certs
      - ./vhost.d:/etc/nginx/vhost.d
      - ./html:/usr/share/nginx/html
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true
```

Repasemos las partes que no son obvias:

- **`ports: 80 y 443`** — este es el único contenedor de todo el servidor que publica puertos al exterior. Todos los demás quedan detrás.
- **`container_name: nginx-proxy`** — fijar el nombre no es cosmético: otras piezas (el compañero de certificados, los comandos de recarga) lo buscan por ese nombre exacto.
- **`/var/run/docker.sock:/tmp/docker.sock:ro`** — el corazón del automatismo. El *socket* de Docker es la API por la que se controla el demonio; montándolo dentro, el proxy puede preguntar "¿qué contenedores hay y con qué variables de entorno?" y suscribirse a los eventos de arranque y parada. **El `:ro` del final (solo lectura) no es opcional**: quien puede escribir en ese socket puede crear contenedores privilegiados y controlar el host entero.
- **`./certs`, `./vhost.d`, `./html`** — directorios del host montados dentro. `certs` guarda los certificados TLS, `vhost.d` permite añadir configuración propia por dominio, y `html` sirve las páginas que usa la validación de certificados. Al vivir en el host, sobreviven a la recreación del contenedor.

Arráncalo:

```bash
docker compose up -d
```

Comprueba que está escuchando y que no hay errores de configuración:

```bash
docker logs nginx-proxy --tail 20
```

En este punto, una petición a la IP del servidor devuelve un `503 Service Temporarily Unavailable`. Eso **es la respuesta correcta**: el proxy funciona, pero todavía no conoce ningún dominio al que enviar la petición.

## Conectar una aplicación

Ahora la parte que hace que todo esto merezca la pena. Una aplicación se registra en el proxy con una variable de entorno y una red. Nada más.

```yaml
services:
  tienda-frontend:
    image: registro.ejemplo.com/tienda-frontend:1.4.0
    container_name: tienda-frontend
    restart: always
    environment:
      - VIRTUAL_HOST=tienda.ejemplo.com
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true
```

Fíjate en lo que **no** hay: ninguna sección `ports:`. La aplicación no publica nada al exterior; solo el proxy lo hace. Desde fuera, el único camino hasta este contenedor pasa por `nginx-proxy`.

`VIRTUAL_HOST=tienda.ejemplo.com` es la declaración completa. Al arrancar el contenedor, el proxy detecta la variable, regenera su configuración y empieza a enviar a este contenedor todas las peticiones cuya cabecera `Host` sea ese dominio.

Los dos requisitos para que funcione, y las dos causas de que no funcione:

1. **La misma red que el proxy.** Si el contenedor no está en `proxy-net`, el proxy no lo alcanza aunque detecte la variable. Se comprueba con `docker network inspect proxy-net`, que lista los contenedores conectados.
2. **Un registro DNS de tipo A** que apunte `tienda.ejemplo.com` a la IP pública del servidor. Sin esto, el navegador ni siquiera llega.

Si la aplicación escucha en un puerto distinto del 80 dentro del contenedor, hay que decírselo:

```yaml
environment:
  - VIRTUAL_HOST=tienda.ejemplo.com
  - VIRTUAL_PORT=3000        # el puerto interno del contenedor
```

`VIRTUAL_PORT` solo hace falta si el contenedor expone varios puertos o si no es el 80. Es el segundo motivo más frecuente de `502 Bad Gateway`.

## Configuración propia por dominio

El automatismo cubre el enrutamiento, pero antes o después necesitas ajustar algo: cabeceras de seguridad, tiempos de espera más largos, compresión. Para eso está `vhost.d`: un fichero **con el nombre exacto del dominio** cuyo contenido se inyecta en el bloque `server` de ese dominio.

```bash
nano /srv/docker/nginx-proxy/vhost.d/tienda.ejemplo.com
```

```nginx
add_header Strict-Transport-Security  "max-age=63072000; includeSubDomains; preload" always;
add_header X-Frame-Options            "SAMEORIGIN" always;
add_header X-Content-Type-Options     "nosniff" always;
add_header Referrer-Policy            "strict-origin-when-cross-origin" always;
add_header Permissions-Policy         "camera=(), microphone=(), geolocation=()" always;

gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000;

proxy_connect_timeout 60s;
proxy_send_timeout    120s;
proxy_read_timeout    120s;
```

Qué consigue cada bloque:

- **`Strict-Transport-Security`** obliga al navegador a usar solo HTTPS con este dominio durante dos años, incluso si el usuario escribe `http://`. Es la cabecera que cierra la ventana de ataque del primer acceso. Cuidado con `preload`: es difícil de revertir, así que actívalo solo cuando estés seguro de que el dominio y todos sus subdominios servirán HTTPS de forma permanente.
- **`X-Frame-Options: SAMEORIGIN`** impide que otro sitio meta tu web en un `iframe` para engañar al usuario (*clickjacking*).
- **`X-Content-Type-Options: nosniff`** evita que el navegador adivine el tipo de un fichero e interprete como script algo que no lo era.
- **`Referrer-Policy`** limita la información de origen que se envía al navegar a otros sitios.
- **`Permissions-Policy`** desactiva cámara, micrófono y geolocalización para el sitio entero.
- **`gzip`** comprime las respuestas de texto. `gzip_min_length 1000` evita comprimir respuestas diminutas, donde el coste de CPU supera el ahorro.
- **`proxy_read_timeout 120s`** sube el límite de espera por defecto (60 s). Es lo que necesitas si alguna operación tarda —generar un informe, procesar una subida grande— y el usuario ve un `504 Gateway Timeout` justo al minuto.

El `always` de las cabeceras es importante: sin él, nginx solo las añade en respuestas correctas (2xx, 3xx), y las páginas de error saldrían sin protección.

Para aplicar cambios en `vhost.d` **no hace falta reiniciar el contenedor**. Basta con pedirle a nginx que recargue, lo que ocurre sin cortar ninguna conexión en curso:

```bash
docker exec nginx-proxy nginx -s reload
```

Antes de recargar, conviene validar la sintaxis; un error deja el proxy con la configuración anterior pero es mejor saberlo:

```bash
docker exec nginx-proxy nginx -t
```

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Además de un fichero por dominio, `vhost.d/default` aplica a todos los dominios a la vez — útil para las cabeceras de seguridad que quieres en todas partes.

## Diagnosticar cuando algo no llega

El proxy da tres respuestas de error muy distintas, y cada una apunta a un sitio:

| Respuesta | Qué significa | Dónde mirar |
|---|---|---|
| `503 Service Temporarily Unavailable` | El proxy no conoce ese dominio | ¿Está `VIRTUAL_HOST` bien escrito? ¿Arrancó el contenedor? |
| `502 Bad Gateway` | Lo conoce pero no puede alcanzarlo | ¿Está en `proxy-net`? ¿Falta `VIRTUAL_PORT`? ¿La app arrancó de verdad? |
| `504 Gateway Timeout` | Lo alcanza pero tarda demasiado | Sube `proxy_read_timeout`, o mira por qué la app va lenta |

La herramienta definitiva es ver la configuración que el proxy ha generado. Ahí está la verdad sobre qué dominios conoce y a dónde envía cada uno:

```bash
docker exec nginx-proxy cat /etc/nginx/conf.d/default.conf | grep -A3 "server_name"
```

```
    server_name tienda.ejemplo.com;
    ...
    upstream tienda.ejemplo.com {
        server 172.19.0.4:80;
```

Si tu dominio no aparece en esa salida, el problema está en la detección (variable o red). Si aparece pero con un `upstream` vacío, el proxy lo conoce pero no encuentra el contenedor en la red.

Y para confirmar quién está conectado a la red:

```bash
docker network inspect proxy-net -f '{{range .Containers}}{{.Name}} {{end}}'
```

```
nginx-proxy tienda-frontend
```

## Buenas prácticas avanzadas

- **Conecta cada aplicación a dos redes, no a una.** El frontend necesita `proxy-net` para recibir tráfico, pero la base de datos no debe estar ahí: todo lo que esté en `proxy-net` es alcanzable por cualquier otro contenedor de esa red. El patrón correcto es una red pública para lo que recibe tráfico y una red interna para el resto, desarrollado en [Redes Docker y exposición de servicios](Redes-Docker-y-Exposicion-de-Servicios.md).
- **Recarga en lugar de reiniciar.** `docker restart nginx-proxy` corta todas las conexiones activas y deja el sitio caído unos segundos; `nginx -s reload` levanta procesos nuevos con la configuración nueva y deja que los antiguos terminen lo que estaban sirviendo. La diferencia se nota cuando alguien está subiendo un fichero grande.
- **Fija la versión de la imagen del proxy.** `nginxproxy/nginx-proxy` sin etiqueta significa `latest`, y una actualización silenciosa puede cambiar el comportamiento del componente por el que pasa el 100 % de tu tráfico. Anclar `nginxproxy/nginx-proxy:1.6` y actualizar a propósito convierte una sorpresa en una decisión.
- **Ten preparada una página de mantenimiento.** El directorio `html` montado permite servir un HTML propio en lugar del `503` desnudo de nginx. Cuesta cinco minutos configurarlo y es la diferencia entre un usuario que cree que el sitio ha desaparecido y uno que sabe que volverá.
- **Vigila el tamaño máximo de subida.** nginx rechaza cuerpos de petición mayores de 1 MB por defecto con un `413 Request Entity Too Large`, y el error se atribuye casi siempre a la aplicación, que nunca llega a ver la petición. Si tu servicio acepta ficheros, `client_max_body_size 50M;` en `vhost.d` es obligatorio — y recuerda que el límite de la aplicación tiene que ser igual o mayor.

## Recursos didácticos

- [Security Headers](https://securityheaders.com/) — pegas tu dominio y te puntúa las cabeceras de seguridad que estás enviando, explicando qué falta y por qué importa. Es la forma directa de comprobar que el fichero `vhost.d` surte efecto.
- [NGINX Playground](https://nginx-playground.wizardzines.com/) — pegas una configuración de nginx y la valida al instante sin necesidad de servidor. Muy útil para el contenido de `vhost.d` antes de recargar en producción.
- [Repositorio de nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) — el README documenta todas las variables (`VIRTUAL_HOST`, `VIRTUAL_PORT`, `VIRTUAL_PATH`, `HTTPS_METHOD`...) y los puntos de extensión de `vhost.d`.

---

*En resumen: nginx-proxy convierte el reparto de tráfico en algo que cada aplicación declara por sí misma con una variable de entorno — el proxy se entera solo mirando los eventos de Docker.*
