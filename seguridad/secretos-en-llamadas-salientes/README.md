# Secretos en llamadas salientes — Guía práctica

Tutorial introductorio sobre **cómo evitar filtrar una credencial cuando tu servidor la usa para llamar a una API externa**. Es el complemento *en runtime* de la gestión de secretos en desarrollo: allí el foco está en guardar bien el secreto (que no acabe en git); aquí, en no dejarlo escapar en el momento de usarlo. Pensado para perfiles full-stack, con ejemplos genéricos en .NET que se entienden sin conocer ningún proyecto concreto.

La idea central en una frase: **un secreto bien guardado se puede filtrar igualmente al usarlo** — por los logs, por un redirect o por una URL de terceros.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Credenciales en Llamadas Salientes](Credenciales-en-Llamadas-Salientes.md) | Las cinco fugas habituales al llamar a una API autenticada (logs, host equivocado, redirect, URL de terceros, canal sin TLS) y cómo evitarlas. |

---

> Relacionado: [Gestión de secretos en desarrollo](../gestion-de-secretos-en-desarrollo/README.md) (guardar el secreto en reposo) y [Autenticación y autorización](../autenticacion-y-autorizacion/README.md) (qué es un token y qué autoriza).
