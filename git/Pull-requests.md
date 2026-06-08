# Pull requests

## ¿Qué es?

Un *pull request* (PR) —o *merge request* en GitLab— es una propuesta de fusión: pides que los cambios de tu rama se revisen y se incorporen a otra rama (normalmente `main`). Es una función de plataformas como GitHub o GitLab, no de Git en sí.

## ¿Por qué existe?

En un equipo, no quieres que cualquiera meta cambios directamente en la rama principal sin que nadie los mire. El pull request crea un espacio para **revisar el código antes de fusionarlo**: tus compañeros pueden comentar línea a línea, sugerir mejoras y aprobar o pedir cambios. Es donde ocurre la conversación sobre el código.

> Si una rama es tu borrador y la fusión es publicarlo, el pull request es el momento de "pasarlo a revisión" antes de que entre en la versión oficial.

## ¿Cuándo y para qué se usa?

- Para **proponer** que tu trabajo (una función, un arreglo) entre en `main`.
- Para que el equipo **revise** el código y deje comentarios.
- Para que se ejecuten **comprobaciones automáticas** (tests, linters) antes de fusionar (ver [GitHub Actions](GitHub-Actions.md)).

## Lo mínimo que necesitas saber

**1. El flujo típico**

```bash
git switch -c funcion-carrito   # creas tu rama
# ...trabajas y haces commits...
git push -u origin funcion-carrito  # la subes al remoto
```

Después, en la web de GitHub/GitLab, pulsas "Create pull request".

**2. Qué contiene un buen PR**

- Un **título claro** y una descripción de qué cambia y por qué.
- Cambios **pequeños y enfocados**: es más fácil revisar 50 líneas que 5.000.

**3. El ciclo de revisión**

Quien revisa deja comentarios; tú respondes haciendo más commits en la misma rama (el PR se actualiza solo). Cuando se aprueba, se fusiona.

**4. Tras fusionar, limpia**

La rama suele borrarse al fusionar el PR, para no acumular ramas viejas.

## Lo que NO hace

- **No es parte de Git** — es una función de la plataforma (GitHub, GitLab...); Git solo conoce ramas y fusiones.
- **No garantiza calidad por sí solo** — es una herramienta para la revisión humana y automática; el criterio sigue siendo del equipo.
- **No fusiona automáticamente** — espera aprobación (y, a menudo, que pasen las comprobaciones).

---

*En resumen: un pull request es el punto de control donde tu trabajo se revisa y se discute antes de entrar en la rama principal — el corazón de la colaboración en equipo.*
