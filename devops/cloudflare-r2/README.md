# Cloudflare R2 — Guía de tecnologías

Cómo guardar y servir ficheros (imágenes, documentos, copias de seguridad) con Cloudflare R2, el almacenamiento de objetos compatible con S3 y sin cargo por tráfico de salida. Va dirigida a perfiles backend que ya han subido ficheros a disco o a algún bucket y quieren decidir cómo organizarlos, protegerlos y entregarlos.

Los ejemplos usan una tienda online con un bucket `tienda-media` servido desde `cdn.ejemplo.com`.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [R2](R2.md) | Crear el bucket y las credenciales, usarlo desde código, las formas de dar acceso a los ficheros (URL prefirmada, dominio personalizado), costes, ciclo de vida y errores frecuentes. |

---

> Piezas relacionadas: [Copias de seguridad](../despliegue-en-vps/Copias-de-Seguridad.md) para sacar los volcados del servidor, y [Caching](../../bases-de-datos/caching/README.md) para el uso de cachés delante de los datos.
