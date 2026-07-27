# HTTPS con Let's Encrypt

## ¿Qué es?

Let's Encrypt es una autoridad de certificación gratuita que emite certificados TLS de forma automática. En un servidor con contenedores, el compañero `acme-companion` se encarga de pedirlos, instalarlos y renovarlos sin que nadie intervenga.

## ¿Por qué existe?

Hasta hace unos años, poner HTTPS en un sitio significaba pagar entre 50 y 300 euros al año, rellenar formularios, esperar la validación y repetir el trámite cada doce meses. El resultado previsible es que la mayor parte de internet iba en HTTP plano, con las contraseñas viajando en claro.

Let's Encrypt eliminó las dos barreras a la vez: el coste (es gratis) y el trámite (es una llamada a una API). Su protocolo, **ACME**, define cómo un programa demuestra automáticamente que controla un dominio y obtiene un certificado por él. Hoy más del 80 % de las páginas web se sirven por HTTPS, y en buena medida es por esto.

> Si te suena la verificación de un dominio en un proveedor de correo —"crea este registro DNS o sube este fichero para demostrar que el dominio es tuyo"—, ACME es exactamente ese mecanismo, pero ejecutado por un programa cada 60 días.

## ¿Cuándo y para qué se usa?

En todo sitio accesible desde internet, sin matices. Un navegador moderno marca como "No seguro" cualquier página HTTP, los buscadores penalizan, y funciones del navegador como la geolocalización o las notificaciones directamente no funcionan sin HTTPS.

Esta ficha da por supuesto que ya tienes un [reverse proxy con nginx-proxy](Reverse-Proxy-con-nginx-proxy.md) funcionando. `acme-companion` es su pieza hermana: no funciona por su cuenta.

---

## Cómo demuestra ACME que el dominio es tuyo

Merece la pena entender el mecanismo, porque casi todos los fallos son fallos de este proceso.

Cuando el cliente pide un certificado para `tienda.ejemplo.com`, Let's Encrypt responde con un reto (*challenge*). El más habitual es **HTTP-01** y funciona así:

1. El cliente recibe un token y lo escribe en un fichero dentro de `/.well-known/acme-challenge/`.
2. Let's Encrypt hace una petición HTTP a `http://tienda.ejemplo.com/.well-known/acme-challenge/<token>` desde sus propios servidores.
3. Si el contenido coincide, queda demostrado que quien pide el certificado controla lo que hay en ese dominio, y emite el certificado.

De ahí salen los tres requisitos, que son los tres motivos por los que esto falla:

- **Un registro DNS de tipo A** que apunte el dominio a la IP pública del servidor. Let's Encrypt resuelve el nombre desde fuera; si apunta a otro sitio, valida en el sitio equivocado.
- **El puerto 80 abierto al mundo.** El reto HTTP-01 llega por HTTP plano. Esta es la razón por la que no se puede cerrar el 80 "porque todo va por HTTPS": sin él no hay emisión ni renovación. En el [firewall](UFW.md), `ufw allow 80/tcp` no es negociable.
- **Que el servidor responda a ese dominio.** El proxy tiene que conocerlo, que es justo lo que hace `VIRTUAL_HOST`.

## Añadir el compañero de certificados

`acme-companion` se añade al mismo `docker-compose.yml` del proxy. Es un segundo contenedor que vigila los eventos de Docker igual que el proxy, pero busca otra variable: `LETSENCRYPT_HOST`.

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

  letsencrypt:
    image: nginxproxy/acme-companion
    container_name: nginx-proxy-letsencrypt
    restart: always
    environment:
      - DEFAULT_EMAIL=admin@ejemplo.com
      - NGINX_PROXY_CONTAINER=nginx-proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./certs:/etc/nginx/certs
      - ./vhost.d:/etc/nginx/vhost.d
      - ./html:/usr/share/nginx/html
      - ./acme:/etc/acme.sh
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true
```

Las partes que importan del servicio nuevo:

- **`DEFAULT_EMAIL`** — Let's Encrypt avisa a esta dirección si un certificado está a punto de caducar sin haberse renovado. Es la señal de alarma que te enteras de que la renovación automática lleva semanas fallando. Pon un buzón que alguien lea.
- **`NGINX_PROXY_CONTAINER=nginx-proxy`** — le dice a quién pedir la recarga tras instalar un certificado. Debe coincidir con el `container_name` del proxy.
- **Los volúmenes compartidos** — `certs`, `vhost.d` y `html` son **los mismos** que monta el proxy. Ese es todo el mecanismo de comunicación entre ambos: el compañero escribe el certificado en `certs` y el fichero del reto en `html`, y el proxy los lee. Si las rutas no coinciden exactamente, cada uno trabaja en su rincón y nada funciona.
- **`./acme:/etc/acme.sh`** — guarda la cuenta ACME y el estado. Sin este volumen, cada recreación del contenedor registra una cuenta nueva y vuelve a pedir todos los certificados desde cero, lo que consume rápidamente el límite de peticiones.

## Pedir un certificado para una aplicación

Igual que el enrutamiento, el certificado se declara desde la aplicación con dos variables más:

```yaml
services:
  tienda-frontend:
    image: registro.ejemplo.com/tienda-frontend:1.4.0
    container_name: tienda-frontend
    restart: always
    environment:
      - VIRTUAL_HOST=tienda.ejemplo.com
      - LETSENCRYPT_HOST=tienda.ejemplo.com
      - LETSENCRYPT_EMAIL=admin@ejemplo.com
    networks:
      - proxy-net

networks:
  proxy-net:
    external: true
```

`VIRTUAL_HOST` y `LETSENCRYPT_HOST` son variables distintas leídas por contenedores distintos —el proxy y el compañero— y por eso hay que poner las dos con el mismo valor. Es un despiste habitual poner solo la primera y preguntarse por qué el sitio va en HTTP.

Al arrancar, la emisión tarda entre 30 segundos y dos minutos. El progreso se ve en los logs del compañero:

```bash
docker logs -f nginx-proxy-letsencrypt
```

```
Creating/renewal tienda.ejemplo.com certificates... (tienda.ejemplo.com)
[Mon Aug 25 10:14:22 UTC 2025] Getting webroot for domain='tienda.ejemplo.com'
[Mon Aug 25 10:14:24 UTC 2025] Verifying: tienda.ejemplo.com
[Mon Aug 25 10:14:27 UTC 2025] Success
[Mon Aug 25 10:14:31 UTC 2025] Cert success.
Reloading nginx proxy (nginx-proxy)...
```

Ese `Cert success` seguido del `Reloading nginx proxy` es la confirmación de que ya hay HTTPS. A partir de ahí, `https://tienda.ejemplo.com` funciona y las peticiones a `http://` se redirigen solas al puerto 443.

Compruébalo desde fuera:

```bash
curl -I https://tienda.ejemplo.com
```

```
HTTP/2 200
server: nginx
strict-transport-security: max-age=63072000
```

Y los certificados quedan en el host, en la carpeta que montaste:

```bash
ls /srv/docker/nginx-proxy/certs/
```

```
tienda.ejemplo.com.crt  tienda.ejemplo.com.key  tienda.ejemplo.com.chain.pem
```

## Un certificado para varios dominios

Si la misma aplicación responde en el dominio con y sin `www`, se listan separados por comas:

```yaml
environment:
  - VIRTUAL_HOST=tienda.ejemplo.com,www.tienda.ejemplo.com
  - LETSENCRYPT_HOST=tienda.ejemplo.com,www.tienda.ejemplo.com
```

Se emite **un solo certificado** que cubre ambos nombres (lo que técnicamente se llama SAN, *Subject Alternative Name*). Los dos dominios necesitan su registro DNS apuntando al servidor: si `www` no está en el DNS, la validación falla y **no se emite ninguno de los dos**, ni siquiera el que sí funcionaba.

## La renovación, y por qué a veces falla en silencio

Los certificados de Let's Encrypt duran **90 días**. El compañero comprueba a diario y renueva cuando quedan menos de 30, así que hay un mes entero de margen para que una renovación fallida se arregle.

El problema es que ese margen se consume sin que nadie mire. Los casos típicos:

- Alguien cerró el puerto 80 en el firewall "porque ya no hacía falta".
- El registro DNS cambió al migrar el dominio a otro proveedor.
- El contenedor del compañero lleva parado desde un despliegue.

Y el síntoma no aparece hasta el día 90, cuando el navegador muestra un aviso de certificado caducado a pantalla completa. Para no llegar ahí, comprueba la fecha de caducidad:

```bash
echo | openssl s_client -connect tienda.ejemplo.com:443 -servername tienda.ejemplo.com 2>/dev/null \
  | openssl x509 -noout -dates
```

```
notBefore=Aug 25 09:14:31 2025 GMT
notAfter=Nov 23 09:14:30 2025 GMT
```

El `-servername` es necesario: sin él, un servidor con varios dominios devuelve el certificado por defecto y estarías comprobando otro.

Para forzar una renovación sin esperar, se reinicia el compañero, que revisa todos los certificados al arrancar:

```bash
docker restart nginx-proxy-letsencrypt
docker logs -f nginx-proxy-letsencrypt
```

## Cuidado con los límites de Let's Encrypt

Let's Encrypt aplica límites por dominio, y el que más duele es **5 certificados idénticos por semana**. Es fácil agotarlo depurando: cada `docker compose up --force-recreate` mientras intentas averiguar por qué falla cuenta como un intento.

Cuando lo alcanzas, el log lo dice claro:

```
too many certificates already issued for exact set of domains: tienda.ejemplo.com
```

Y a partir de ahí solo queda esperar una semana. Para evitarlo, mientras depuras usa el **entorno de pruebas** (*staging*), que tiene límites muchísimo más altos:

```yaml
environment:
  - ACME_CA_URI=https://acme-staging-v02.api.letsencrypt.org/directory
```

Los certificados de staging no los reconoce el navegador —verás un aviso de seguridad—, pero confirman que todo el mecanismo funciona. Cuando la emisión funcione en staging, quita esa variable, borra los certificados de prueba de `./certs` y recrea el contenedor para pedir el bueno.

## Errores frecuentes

| Mensaje en el log | Causa |
|---|---|
| `Timeout during connect` | El puerto 80 está cerrado en el firewall |
| `DNS problem: NXDOMAIN looking up A` | No existe el registro DNS, o no ha propagado |
| `Invalid response ... 404` | El proxy no conoce el dominio: falta `VIRTUAL_HOST` |
| `too many certificates already issued` | Límite semanal agotado; usa staging |
| Nada en el log, no pasa nada | Falta `LETSENCRYPT_HOST` en la aplicación |

Antes de investigar por dentro, comprueba lo de fuera. Que el DNS resuelva a la IP correcta:

```bash
dig +short tienda.ejemplo.com
```

```
203.0.113.10
```

Y que el puerto 80 responda desde internet (no desde el propio servidor, que siempre funciona):

```bash
curl -I http://tienda.ejemplo.com/.well-known/acme-challenge/test
```

Un `404` es buena señal: significa que la petición llegó al proxy. Un *timeout* significa que el firewall o el DNS están mal.

## Buenas prácticas avanzadas

- **Monitoriza la caducidad desde fuera del servidor.** Si el aviso depende del propio servidor, un fallo que tumbe el servidor se lleva también el aviso. Un servicio externo de comprobación de certificados, o un simple script en otra máquina que ejecute el `openssl s_client` de arriba y avise cuando queden menos de 20 días, cubre el caso real: la renovación lleva mes y medio fallando y nadie lo sabe.
- **Usa el reto DNS-01 para dominios comodín o servicios internos.** HTTP-01 exige que el dominio sea alcanzable desde internet por el puerto 80, lo que descarta un servicio que solo vive detrás de la [VPN](WireGuard.md). El reto DNS-01 valida creando un registro TXT mediante la API de tu proveedor de DNS, no necesita puertos abiertos y es el único que permite certificados comodín (`*.ejemplo.com`).
- **Haz copia de la carpeta `acme`, no solo de `certs`.** Los certificados se pueden volver a emitir, pero la cuenta ACME y su clave viven en `/etc/acme.sh`. Perderla no es grave, pero obliga a registrar una cuenta nueva y reemitirlo todo de golpe — justo cuando estás restaurando tras un desastre y el límite semanal es lo último que necesitas.
- **Activa HSTS solo cuando el HTTPS sea estable.** La cabecera `Strict-Transport-Security` (ver [el fichero `vhost.d`](Reverse-Proxy-con-nginx-proxy.md)) hace que el navegador se niegue a conectar por HTTP durante el tiempo indicado, y esa decisión queda cacheada en el navegador del usuario. Si activas HSTS con `max-age` de dos años y luego el certificado falla, los usuarios que ya visitaron el sitio no pueden entrar de ninguna manera. Empieza con `max-age=300` y súbelo cuando lleves semanas sin incidencias.
- **No reutilices el mismo dominio en varios servidores a la vez.** Durante una migración es tentador tener el servidor viejo y el nuevo respondiendo al mismo nombre. Con HTTP-01 esto rompe la validación de forma intermitente: Let's Encrypt consulta el DNS y puede acabar en cualquiera de los dos, y solo uno tiene el token. Migra el DNS de una vez, o valida por DNS-01.

## Recursos didácticos

- [SSL Labs Server Test](https://www.ssllabs.com/ssltest/) — analiza tu dominio desde fuera y te da una nota de la A+ a la F, con el detalle de la cadena de certificados, los protocolos y los cifrados aceptados. Es el examen estándar del sector y muy didáctico para entender qué hay detrás del candado.
- [Let's Debug](https://letsdebug.net/) — escribes tu dominio y simula la validación de Let's Encrypt, diciéndote exactamente qué fallaría y por qué (DNS, puerto 80, redirecciones raras). Es la primera parada cuando la emisión no funciona.
- [Documentación de límites de Let's Encrypt](https://letsencrypt.org/docs/rate-limits/) — conviene leerla una vez antes de empezar a depurar en producción.

---

*En resumen: Let's Encrypt convierte el HTTPS en dos variables de entorno — a cambio de mantener el puerto 80 abierto y el DNS apuntando donde debe, para siempre.*
