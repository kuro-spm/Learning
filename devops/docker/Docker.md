# Docker

## ¿Qué es?

Docker es una plataforma de **contenedores** que permite empaquetar una aplicación junto con todo lo que necesita para ejecutarse —sistema operativo base, dependencias, configuración— dentro de una unidad portable y reproducible llamada contenedor.

## ¿Por qué existe?

El problema clásico del desarrollo de software: *"en mi máquina funciona"*. Una aplicación puede funcionar perfectamente en el ordenador de un desarrollador y fallar en el servidor de producción porque las versiones de las herramientas son distintas, o porque hay una librería del sistema que en un entorno está instalada y en el otro no.

Docker resuelve esto garantizando que la aplicación siempre se ejecuta en exactamente el mismo entorno, da igual dónde se despliegue.

> Si ya conoces las máquinas virtuales (VMware, VirtualBox), piensa en Docker como una versión mucho más ligera: en lugar de virtualizar hardware completo, comparte el kernel del sistema operativo del host pero aísla el proceso y su sistema de archivos.

## ¿Cuándo y para qué se usa?

- **Despliegue de aplicaciones**: Empaquetar un servidor web (Node.js, .NET, Python…) para que cualquier servidor pueda ejecutarlo con un solo comando.
- **Entornos de desarrollo reproducibles**: Que todo el equipo trabaje con la misma versión de la base de datos, el mismo runtime, la misma configuración, sin instalar nada en el sistema anfitrión.
- **Microservicios**: Ejecutar varios servicios independientes (API, base de datos, cola de mensajes) en la misma máquina sin que interfieran entre sí.
- **CI/CD**: Correr los tests en un entorno limpio y reproducible en cada pull request.

## Lo mínimo que necesitas saber

**Conceptos clave antes de empezar**

Tres conceptos que aparecen en todo momento:

- **Imagen**: La "receta" de un contenedor. Es un archivo inmutable que describe el sistema operativo base, las dependencias y el código de la aplicación. Se construye a partir de un `Dockerfile`.
- **Contenedor**: Una instancia en ejecución de una imagen. Es como "ejecutar" la receta: puede arrancar, pararse y eliminarse. De la misma imagen puedes crear cientos de contenedores.
- **Dockerfile**: El archivo de texto que define cómo construir una imagen, instrucción por instrucción.

---

**1. Cómo se escribe un Dockerfile a mano**

Un `Dockerfile` es un archivo de texto sin extensión que vive en la raíz del proyecto. Cada línea es una instrucción que Docker ejecuta en orden al construir la imagen.

Ejemplo para una API de .NET:

```dockerfile
# Imagen base con el SDK de .NET para compilar
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /app

# Copiar el archivo de proyecto y restaurar dependencias
COPY *.csproj .
RUN dotnet restore

# Copiar el resto del código y compilar
COPY . .
RUN dotnet publish -c Release -o /out

# Imagen final más ligera (solo runtime, sin SDK)
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /out .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MiApi.dll"]
```

Las instrucciones más frecuentes:

| Instrucción | Qué hace |
|---|---|
| `FROM` | Define la imagen base de la que parte todo |
| `WORKDIR` | Establece el directorio de trabajo dentro del contenedor |
| `COPY` | Copia archivos del host al contenedor |
| `RUN` | Ejecuta un comando durante la construcción de la imagen |
| `EXPOSE` | Documenta en qué puerto escucha la aplicación |
| `ENTRYPOINT` | El comando que se ejecuta al arrancar el contenedor |

---

**2. Comandos esenciales**

```bash
# Construir una imagen a partir del Dockerfile del directorio actual
docker build -t mi-api:latest .

# Ejecutar un contenedor de esa imagen
# -p mapea el puerto 8080 del host al 8080 del contenedor
docker run -p 8080:8080 mi-api:latest

# Ver los contenedores en ejecución
docker ps

# Detener un contenedor
docker stop <id-del-contenedor>

# Ver las imágenes locales
docker images

# Eliminar un contenedor parado
docker rm <id-del-contenedor>
```

---

**3. Docker Compose: varios servicios a la vez**

Cuando la aplicación necesita más de un servicio (por ejemplo, una API más una base de datos), se usa `docker-compose.yml` para definir y arrancar todo con un solo comando.

```yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: usuario
      POSTGRES_PASSWORD: contraseña
      POSTGRES_DB: miapp
    volumes:
      - datos-postgres:/var/lib/postgresql/data

volumes:
  datos-postgres:
```

```bash
# Arrancar todos los servicios
docker compose up

# Arrancarlos en segundo plano
docker compose up -d

# Pararlos y eliminar los contenedores
docker compose down
```

---

**4. Cómo genera Claude un Dockerfile automáticamente**

Cuando se le pide a Claude que cree un `Dockerfile` o un `docker-compose.yml`, el proceso no consiste en escribir una plantilla genérica y rellenar huecos. Estos son los pasos reales que sigue:

**Paso 1 — Identificar el tipo de aplicación**

Lo primero es entender qué hay que contenerizar. Claude lee los archivos del proyecto para detectar:
- El lenguaje y runtime (`.csproj` → .NET, `package.json` → Node.js, `requirements.txt` → Python…).
- La versión concreta del runtime que usa el proyecto (evita poner una versión aleatoria que rompa la aplicación).
- Si es una API, una aplicación web, un worker, una función…

**Paso 2 — Buscar las dependencias externas**

Claude revisa la configuración del proyecto para detectar qué servicios externos necesita: ¿usa una base de datos?, ¿qué motor?, ¿un sistema de caché como Redis?, ¿una cola de mensajes? Estos servicios acaban como entradas adicionales en `docker-compose.yml`.

**Paso 3 — Elegir las imágenes base correctas**

Conociendo el runtime y la versión, Claude elige las imágenes oficiales adecuadas. Para .NET, por ejemplo, hay que distinguir entre la imagen con el SDK completo (necesaria para compilar) y la imagen de solo runtime (más ligera, para producción). Usar la imagen correcta reduce el tamaño final de la imagen y la superficie de ataque de seguridad.

**Paso 4 — Decidir si usar multi-stage build**

Si el proyecto necesita compilarse antes de ejecutarse (como en .NET, Go o Java), Claude estructura el Dockerfile en dos etapas:
1. Una etapa de *build* con el SDK completo que compila el código.
2. Una etapa final con solo el runtime, que copia únicamente el artefacto compilado.

Esto produce imágenes de producción mucho más pequeñas (a veces 10 veces más ligeras que sin esta técnica).

**Paso 5 — Optimizar el orden de las capas**

Docker cachea cada instrucción del Dockerfile como una "capa". Si una capa cambia, todas las siguientes se reconstruyen desde cero. Por eso Claude sigue este orden deliberado:

1. Copiar primero los archivos que cambian poco (como el `.csproj` o el `package.json`).
2. Ejecutar la restauración/instalación de dependencias.
3. Copiar el código fuente (que cambia con frecuencia) al final.

Así, si solo modificas código pero no añades paquetes nuevos, Docker reutiliza la capa de dependencias y la construcción es mucho más rápida.

**Paso 6 — Configurar puertos, variables de entorno y volúmenes**

Claude revisa la configuración de la aplicación para:
- Exponer el puerto correcto con `EXPOSE` y su mapeo en `docker-compose.yml`.
- Identificar qué valores sensibles (contraseñas, cadenas de conexión) deben pasarse como variables de entorno, no escritos directamente en el archivo.
- Definir volúmenes para los datos que deben persistir entre reinicios del contenedor (carpetas de base de datos, archivos subidos por usuarios…).

**Paso 7 — Escribir el archivo con comentarios explicativos**

Por último, Claude escribe el archivo con comentarios en los bloques no obvios y explica qué hace cada parte para que puedas entenderlo, modificarlo y aprenderlo, no solo copiarlo.

## Lo que NO hace

- **No virtualiza hardware** — comparte el kernel del sistema operativo del host, así que un contenedor Linux no corre de forma nativa en Windows sin una capa de traducción (WSL2 o una VM).
- **No reemplaza la gestión de secretos** — las variables de entorno en `docker-compose.yml` no son seguras para producción real; herramientas como Vault o los secrets de Kubernetes son para eso.
- **No gestiona múltiples máquinas** — eso es Kubernetes. Docker y Docker Compose trabajan en una sola máquina.
- **No garantiza seguridad automática** — un contenedor mal configurado puede exponer la red del host o escalar privilegios; la configuración hay que revisarla.

## Buenas prácticas avanzadas

- **Fija la versión exacta de la imagen base, nunca `latest`** — `FROM node:latest` (o incluso `node:20`) apunta a imágenes distintas según el día en que construyas, así que dos builds del mismo Dockerfile pueden producir resultados diferentes. Usa etiquetas concretas (`node:20.11-alpine`) y, para despliegues críticos, el digest inmutable (`node:20.11-alpine@sha256:...`), que no puede cambiar aunque alguien re-publique la etiqueta.
- **El `.dockerignore` importa tanto como el Dockerfile** — sin él, `COPY . .` arrastra `node_modules`, `.git`, artefactos de build y hasta archivos `.env` con secretos al contexto de build. No solo engorda la imagen: cualquier archivo tocado invalida la caché de esa capa, y un `.env` copiado por accidente queda incrustado para siempre en las capas de la imagen, aunque luego lo borres con otro `RUN`.
- **Usa siempre la forma *exec* en `ENTRYPOINT`/`CMD`** — `ENTRYPOINT dotnet MiApi.dll` (forma *shell*) envuelve el proceso en `/bin/sh -c`, y ese `sh` se convierte en PID 1: tu aplicación nunca recibe el `SIGTERM` de `docker stop`, así que no cierra conexiones ni termina peticiones en curso; Docker espera 10 segundos y la mata a la fuerza. La forma *exec* (`ENTRYPOINT ["dotnet", "MiApi.dll"]`) hace que la app sea PID 1 y reciba las señales directamente.
- **No ejecutes como root dentro del contenedor** — casi todas las imágenes oficiales arrancan como root por defecto. Si alguien explota una vulnerabilidad de tu app, es root dentro del contenedor, y cualquier volumen montado o descuido de configuración convierte eso en un problema del host. Crea un usuario sin privilegios en el Dockerfile y actívalo con `USER` antes del `ENTRYPOINT` (las imágenes de .NET 8+ ya traen uno preparado: `USER app`).
- **`depends_on` no espera a que el servicio esté listo, solo a que arranque** — el error clásico en Compose: la API arranca antes de que PostgreSQL acepte conexiones y falla al conectar. La solución es declarar un `healthcheck` en el servicio de la base de datos y usar la forma larga `depends_on: { db: { condition: service_healthy } }`, que sí espera al chequeo real.

---

*En resumen: Docker empaqueta tu aplicación y todo lo que necesita en una caja estandarizada que se ejecuta igual en cualquier sitio — y cuando Claude crea esa caja por ti, lee tu proyecto, elige las piezas correctas y ordena las instrucciones para que la construcción sea lo más rápida y ligera posible.*
