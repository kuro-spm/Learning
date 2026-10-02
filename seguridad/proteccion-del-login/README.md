# Protección del login — Guía de tecnologías

Dos guías para proteger el endpoint de login de una API ASP.NET Core frente a la fuerza bruta y a quien intenta tumbarlo, y para que esa protección siga funcionando cuando la API vive detrás de un reverse proxy. Pensado para quien ya programa APIs y no ha tenido que ponerle freno a un login en producción.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Limitar-intentos-de-login](Limitar-intentos-de-login.md) | Punto de partida: límite por IP y por cuenta, bloqueo con backoff, y cómo evitar que el propio contador agote la memoria. |
| 2 | [IP-real-detras-de-un-reverse-proxy](IP-real-detras-de-un-reverse-proxy.md) | Sin la IP real del cliente, el límite por IP de la guía anterior deja de funcionar detrás de nginx: cómo recuperarla sin abrir un bypass. |

---

> Fundamentos relacionados: [Contraseñas vs tokens de sesión](../algoritmos-de-hash/Contrasenas-Vs-Tokens-De-Sesion.md), [Funciones de derivación de claves](../algoritmos-de-hash/Funciones-De-Derivacion-De-Claves.md), [Fail2ban](../../devops/despliegue-en-vps/Fail2ban.md) y [Reverse proxy con nginx-proxy](../../devops/despliegue-en-vps/Reverse-Proxy-con-nginx-proxy.md).
