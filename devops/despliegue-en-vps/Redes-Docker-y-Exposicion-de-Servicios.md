# Redes Docker y exposición de servicios

## ¿Qué es?

Una red de Docker es una red virtual privada que conecta contenedores entre sí. Decidir qué contenedor está en qué red —y cuál publica puertos al exterior— es lo que determina qué partes de tu aplicación son alcanzables desde internet y cuáles no.

## ¿Por qué existe?

Una aplicación típica tiene tres piezas: un frontend que ve el usuario, una API que hace el trabajo y una base de datos. Solo la primera necesita ser pública. La API debería recibir peticiones únicamente a través del proxy, y la base de datos no debería aceptar conexiones de nadie que no sea la API.

Sin redes, todos los contenedores de un servidor comparten la red por defecto y se ven entre sí. Con puertos publicados, cualquiera de ellos puede quedar expuesto a internet por accidente. Las redes de Docker permiten dibujar esas fronteras de forma explícita: dos contenedores que no comparten red **no pueden hablarse**, punto. No es una regla de firewall que se pueda saltar; simplemente no hay ruta entre ellos.

> Si has diseñado alguna arquitectura por capas donde la capa de presentación no puede llamar directamente a la base de datos, aquí es lo mismo pero a nivel de red: la separación no depende de que nadie escriba el `using` equivocado, sino de que la conexión no existe.

## ¿Cuándo y para qué se usa?

Cada vez que despliegas más de un contenedor. El caso central es el que acabamos de describir: separar lo público de lo privado en una aplicación web. Pero también aparece al alojar varias aplicaciones independientes en el mismo servidor, donde lo que quieres es que la base de datos de una no sea alcanzable desde los contenedores de la otra.

Esta ficha se apoya en el [reverse proxy](Reverse-Proxy-con-nginx-proxy.md): la red `proxy-net` que allí se crea es una de las dos piezas del patrón que veremos.

---

## Lo primero: descubrimiento por nombre

Los contenedores de una misma red se resuelven por su nombre. Docker ejecuta un DNS interno que traduce el nombre del servicio a su IP, que puede cambiar en cada arranque.

Con este `docker-compose.yml`:

```yaml
services:
  tienda-api:
    image: registro.ejemplo.com/tienda-api:1.4.0
    networks:
      - internal-net

  tienda-db:
    image: postgres:16
    networks:
      - internal-net

networks:
  internal-net:
```

la API se conecta a la base de datos con la cadena `Host=tienda-db;Port=5432;Database=tienda`. El nombre `tienda-db` lo resuelve Docker.

Compruébalo desde dentro de un contenedor:

```bash
docker exec tienda-api getent hosts tienda-db
```

```
172.20.0.3      tienda-db
```

Dos consecuencias prácticas de esto:

1. **No hace falta publicar puertos para que dos contenedores se comuniquen.** La sección `ports:` sirve exclusivamente para exponer un puerto al **host** y, a través de él, a internet. Entre contenedores de la misma red, todos los puertos son alcanzables sin declarar nada.
2. **El puerto es el interno del contenedor.** Aunque publiques `"8080:5432"`, otro contenedor sigue conectándose al 5432, no al 8080. El mapeo solo afecta al host.

## `expose` y `ports` no son lo mismo

Esta confusión aparece en casi todos los `docker-compose.yml` de internet:

```yaml
services:
  tienda-api:
    expose:
      - "3000"        # documentación, no hace nada funcional
    ports:
      - "3000:3000"   # ⚠️ abre el puerto 3000 del servidor a internet
```

- **`expose`** es puramente informativo entre contenedores de la misma red: el puerto ya era alcanzable con o sin él. Se puede omitir casi siempre.
- **`ports`** publica el puerto en la interfaz de red del **host**. Con el formato `"3000:3000"`, Docker escucha en **todas** las interfaces, incluida la IP pública.

Y aquí viene lo importante: como se explica en la ficha de [UFW](UFW.md), Docker escribe sus reglas por debajo de las del firewall, así que `ports:` abre el puerto **aunque UFW diga que está bloqueado**. Un `ports: - "5432:5432"` en una base de datos es una base de datos accesible desde internet, por mucho firewall que tengas.

Si de verdad necesitas alcanzar un puerto desde el propio servidor pero no desde fuera, ata la publicación a localhost:

```yaml
ports:
  - "127.0.0.1:5432:5432"   # solo desde el propio servidor
```

Desde tu portátil llegarías con un túnel SSH, que reutiliza la conexión ya autenticada:

```bash
ssh -L 5432:localhost:5432 tienda
# ahora localhost:5432 en tu portátil es la base de datos del servidor
```

## El patrón de dos redes

Este es el diseño que resuelve el problema completo: una red para lo que recibe tráfico de fuera y otra para las conversaciones internas.

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
      - proxy-net       # para que el proxy lo alcance
      - internal-net    # para hablar con la API

  tienda-api:
    image: registro.ejemplo.com/tienda-api:1.4.0
    container_name: tienda-api
    restart: always
    # sin VIRTUAL_HOST: no es alcanzable desde fuera
    networks:
      - internal-net

  tienda-db:
    image: postgres:16
    container_name: tienda-db
    restart: always
    volumes:
      - tienda-db-data:/var/lib/postgresql/data
    networks:
      - internal-net

networks:
  proxy-net:
    external: true
  internal-net:
    internal: true

volumes:
  tienda-db-data:
```

Lo que consigue cada decisión:

- **El frontend está en las dos redes.** Necesita `proxy-net` para que nginx-proxy le entregue las peticiones, e `internal-net` para llamar a la API. Es el único punto de contacto entre ambos mundos.
- **La API y la base de datos solo están en `internal-net`.** No hay ninguna ruta desde el proxy —ni desde internet— hasta ellas. Aunque alguien adivinase el nombre del contenedor, no hay red por la que llegar.
- **`internal: true` en la red interna.** Esto merece explicación aparte, abajo.
- **`external: true` en `proxy-net`** significa "esta red la creé yo a mano, no la gestiones tú".
- **Ningún `ports:` en todo el fichero.** El único contenedor del servidor que publica puertos es el proxy.

### Qué hace exactamente `internal: true`

Una red normal de Docker permite a sus contenedores **salir** a internet, aunque nadie pueda entrar. Con `internal: true`, se corta también la salida: los contenedores de esa red no tienen ruta hacia el exterior.

Se comprueba fácil:

```bash
docker exec tienda-db ping -c1 1.1.1.1
```

```
ping: connect: Network is unreachable
```

Es una medida de contención valiosa: si alguien logra ejecutar código en la base de datos, no puede descargar herramientas ni enviar los datos a ningún sitio.

Pero tiene un coste que hay que conocer antes de activarlo, porque el fallo despista mucho. Un contenedor en una red interna **no puede**:

- Descargar dependencias o actualizaciones al arrancar.
- Llamar a APIs externas (una pasarela de pago, un servicio de correo, un webhook).
- Resolver nombres de dominio públicos.

Si la API tiene que llamar a un servicio externo, no puede estar solo en una red interna. El reparto habitual es dejar `internal: true` para la red de la base de datos y usar una red normal para la capa de aplicación.

## Enrutar la API por el mismo dominio

Un montaje muy común es servir el frontend en `tienda.ejemplo.com` y la API en `tienda.ejemplo.com/api/`, en lugar de darle un subdominio propio. Ventajas: un solo certificado, un solo dominio y **cero problemas de CORS**, porque para el navegador todo es el mismo origen.

El enrutamiento se hace en el proxy, con un fichero en `vhost.d` con el nombre del dominio:

```bash
nano /srv/docker/nginx-proxy/vhost.d/tienda.ejemplo.com
```

```nginx
location /api/ {
    proxy_pass         http://tienda-api:3000/;
    proxy_set_header   Host               $host;
    proxy_set_header   X-Real-IP          $remote_addr;
    proxy_set_header   X-Forwarded-For    $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto  $scheme;
}
```

Qué hace cada línea:

- **`proxy_pass http://tienda-api:3000/`** — reenvía al contenedor por su nombre de red. Fíjate en la **barra final**: con ella, una petición a `/api/products` llega a la API como `/products`; sin ella, llegaría como `/api/products`. Es la causa clásica de un 404 que no se entiende.
- **`Host $host`** — conserva el dominio original. Sin esto, la API recibiría `tienda-api` como host y cualquier URL que genere saldría mal.
- **`X-Real-IP` y `X-Forwarded-For`** — transmiten la IP real del cliente. Sin ellas, todos los accesos aparecen en los logs de la aplicación con la IP del proxy, lo que arruina cualquier análisis y hace que un [Fail2ban](Fail2ban.md) sobre esos logs bloquee al proxy.
- **`X-Forwarded-Proto $scheme`** — informa de que la petición original era HTTPS. El proxy habla con la API por HTTP plano dentro de la red interna, así que sin esta cabecera la aplicación cree que está sirviendo HTTP y genera enlaces `http://` o entra en bucles de redirección.

Para que el proxy alcance `tienda-api`, hay un requisito que se pasa por alto: **el contenedor de la API tiene que estar en `proxy-net`**, porque es el proxy quien abre la conexión. En el ejemplo anterior no lo estaba, así que hay que añadírsela:

```yaml
  tienda-api:
    networks:
      - internal-net
      - proxy-net      # necesario para que el proxy pueda enrutar /api/
```

Sigue sin tener `VIRTUAL_HOST`, así que no responde a ningún dominio propio: solo es alcanzable por la ruta que el proxy enruta explícitamente.

Aplica el cambio recargando nginx, sin cortar conexiones:

```bash
docker exec nginx-proxy nginx -t && docker exec nginx-proxy nginx -s reload
```

## Comprobar quién ve a quién

La forma directa de auditar el resultado es listar los contenedores de cada red:

```bash
docker network inspect internal-net -f '{{range .Containers}}{{.Name}} {{end}}'
```

```
tienda-frontend tienda-api tienda-db
```

Y la prueba definitiva es intentar la conexión desde donde no debería funcionar. Desde el proxy, la base de datos tiene que ser inalcanzable:

```bash
docker exec nginx-proxy wget -qO- --timeout=3 http://tienda-db:5432
```

```
wget: bad address 'tienda-db'
```

Ese `bad address` es la respuesta correcta: el DNS de Docker no resuelve el nombre porque no comparten red. Si en lugar de eso obtienes una conexión rechazada o un timeout, significa que **sí** hay ruta y la separación no está bien hecha.

## Buenas prácticas avanzadas

- **Trata cualquier `ports:` como una decisión de seguridad.** En un servidor con reverse proxy, el único contenedor que debería publicar puertos es el proxy. Cada `ports:` adicional es un agujero en el firewall que UFW no puede cerrar. Cuando revises un `docker-compose.yml` ajeno, busca esa palabra primero.
- **No dependas de la red para autenticar.** Que la base de datos solo sea alcanzable desde `internal-net` no es motivo para dejarla sin contraseña o con una débil. El aislamiento de red y las credenciales son capas independientes: la primera cae en cuanto alguien logra ejecutar código en cualquier contenedor de esa red. Se ve mucho un Redis o un Mongo sin autenticación "porque está en la red interna", y es el primer sitio donde mira quien entra.
- **Fija la subred si vas a interconectar con VPN u otras redes.** Docker asigna rangos de `172.17.0.0/16` en adelante de forma automática, y antes o después uno de ellos choca con la red corporativa o con la de la [VPN](WireGuard.md). El síntoma es desconcertante: una IP interna que deja de ser alcanzable desde el servidor. Declarar `ipam.config.subnet` explícitamente en las redes evita el conflicto y hace la configuración reproducible.
- **Una red interna por aplicación, no una compartida.** Si dos aplicaciones distintas comparten `internal-net`, la base de datos de una es alcanzable desde los contenedores de la otra. Compose ya ayuda: crea las redes con el nombre del proyecto por delante, así que dos carpetas distintas obtienen redes distintas — siempre que no las declares `external`.
- **Comprueba las cabeceras de proxy en la aplicación, no solo en nginx.** Enviar `X-Forwarded-For` no sirve de nada si el framework no está configurado para confiar en ella. En ASP.NET Core hace falta `UseForwardedHeaders`, en Express `app.set('trust proxy', 1)`. Sin ese paso, la aplicación sigue registrando la IP del proxy y generando URLs con el esquema equivocado — y el fallo es silencioso.

## Recursos didácticos

- [Docker networking tutorial oficial](https://docs.docker.com/engine/network/) — la referencia de los tipos de red (bridge, host, overlay, macvlan) con ejemplos ejecutables de cada uno.
- [Play with Docker](https://labs.play-with-docker.com/) — permite montar el patrón de dos redes en cinco minutos y comprobar de verdad qué contenedor alcanza a cuál, que es la única forma de que el concepto se fije.
- [Subnet Calculator](https://www.subnet-calculator.com/cidr.php) — útil al fijar subredes propias y comprobar que no solapan con la red de la oficina ni con la de la VPN.

---

*En resumen: dos contenedores que no comparten red no pueden hablarse — usa una red pública para lo que recibe tráfico, otra interna para el resto, y que el único `ports:` del servidor sea el del proxy.*
