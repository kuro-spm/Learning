# Por qué los secretos no van a git

## Qué es un "secreto"

Un **secreto** es cualquier dato que da acceso o capacidad y que no debería conocer nadie salvo quien lo necesita: claves de API de servicios externos, contraseñas de base de datos, claves de cifrado, tokens de firma, cadenas de conexión con credenciales dentro.

No es lo mismo que **configuración**. La URL de un endpoint, el nombre de un *deployment* o un número de puerto son configuración: identifican *qué* usas, no *quién* eres. La configuración puede vivir tranquilamente en el repositorio; el secreto no.

> Regla rápida: si con ese dato alguien puede **gastar dinero, leer datos o suplantarte**, es un secreto.

## Por qué git es una trampa

Meter un secreto en el repositorio parece inofensivo ("es solo mi repo privado"), pero tiene tres problemas que no se ven a primera vista:

1. **El historial es para siempre.** Aunque borres la clave en un commit posterior, sigue estando en la historia (`git log -p`, `git show <sha-viejo>`). Quien clone el repo se lleva *todos* los commits. Borrarla de verdad exige reescribir la historia (`git filter-repo`) y rotar la clave igualmente.
2. **El repo se copia y se comparte más de lo que crees.** Forks, clones en portátiles, backups, la caché del sistema de CI, un compañero nuevo, un contratista. Cada copia es una copia del secreto.
3. **"Privado" hoy no es "privado" siempre.** Repos que se hacen públicos por error, permisos mal puestos, una plataforma comprometida. Un secreto en git asume que el repositorio nunca se filtrará — mala apuesta.

## La idea de fondo: dominios de confianza separados

La buena práctica es mantener el secreto en un **dominio de confianza distinto** al del código:

- El **código** vive en git, lo lee mucha gente, se copia por todas partes.
- El **secreto** vive en otro sitio (variable de entorno, almacén de secretos del sistema operativo, un *vault* gestionado) al que solo accede el proceso que lo necesita, en la máquina donde corre.

Así, **quien lee el código no obtiene el secreto**, y **quien roba una copia del repo tampoco**. Si el mismo dato (código + llave) viaja junto, esa separación desaparece y cualquier fuga los expone a la vez.

## Cómo se materializa esto

- **En desarrollo:** *user-secrets* de .NET (ver la siguiente ficha) — un fichero fuera del árbol del repo, en tu perfil de usuario.
- **En producción:** variables de entorno inyectadas por la plataforma, o un gestor de secretos (Key Vault, Secrets Manager, Vault).
- **En el repo:** solo la *forma* de la configuración (claves vacías o de ejemplo en `appsettings.json`) y, como mucho, identificadores que no son secretos.
- **Red de seguridad:** un `.gitignore` que excluya `.env` y similares, y un escáner de secretos en CI para cazar los que se cuelen.

## Lo mínimo que hay que recordar

- Secreto ≠ configuración. Ante la duda, trátalo como secreto.
- Nunca `git add` de una clave real, ni siquiera "temporalmente".
- Si un secreto tocó git aunque sea un commit, considéralo **comprometido** y **rótalo**.
- El objetivo no es esconder el secreto en el código, es **sacarlo del código**.
