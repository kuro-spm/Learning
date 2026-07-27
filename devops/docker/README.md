# Docker — Guía de tecnologías

Cómo funciona la contenedorización y cómo se usa en el día a día de un equipo backend: escribir un `Dockerfile` que aproveche el caché de capas, publicar la imagen resultante en un registro desde CI, y levantar infraestructura real y desechable en los tests.

Está escrita para perfiles backend junior-medio que programan a diario pero pueden no haber usado Docker más allá de un `docker compose up` copiado de algún sitio. No presupone conocimientos de administración de sistemas: cada concepto se explica desde el problema que resuelve, con comandos ejecutables y la salida que devuelven.

Las tres fichas usan el mismo ejemplo: una tienda online con una API .NET llamada `tienda-api` y su base de datos PostgreSQL.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Docker](Docker.md) | El punto de entrada: qué es un contenedor, el modelo de capas y cómo escribir un `Dockerfile` multi-stage que no tarde diez minutos en construirse. |
| 2 | [GitHub Container Registry](GitHub-Container-Registry.md) | Dónde viven las imágenes una vez construidas: cómo se publica desde GitHub Actions y por qué la etiqueta `latest` no vale para producción. |
| 3 | [Testcontainers](Testcontainers.md) | Docker al servicio de los tests: una base de datos real y limpia por ejecución, en lugar de mocks que mienten. |

---

## Cómo encaja con el resto de colecciones

Docker aparece en tres momentos distintos del ciclo de vida, y cada uno tiene su colección:

| Momento | Qué necesitas | Dónde está |
|---|---|---|
| Escribir la imagen | `Dockerfile`, capas, multi-stage | [Docker](Docker.md), aquí |
| Publicarla | Registro, etiquetado, CI | [GitHub Container Registry](GitHub-Container-Registry.md), aquí |
| Probarla | Infraestructura desechable | [Testcontainers](Testcontainers.md), aquí |
| Ejecutarla en producción | Instalación, redes, volúmenes, proxy | [Despliegue en un VPS](../despliegue-en-vps/README.md) |

Esa última fila es la frontera de esta colección: aquí se cubre **la imagen**; cómo se opera un servidor que ejecuta contenedores —rotación de logs, políticas de reinicio, redes internas, permisos de volúmenes, HTTPS— está en la colección de despliegue.

---

> Piezas relacionadas: [Despliegue en un VPS](../despliegue-en-vps/README.md) para llevar la imagen a producción, [CI/CD](../ci-cd/README.md) para automatizar la construcción y publicación, [Testing en .NET](../../testing/testing-dotnet/README.md) para el resto del panorama de pruebas, y [Observabilidad](../observabilidad/README.md) porque en un contenedor lo correcto es escribir los logs a la salida estándar y desentenderse del destino.
