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

**5. Los checks: robots que revisan antes que las personas**

Al abrir (o actualizar) un PR, la plataforma puede lanzar automáticamente los *workflows* de CI — compilar, pasar tests, lint (ver [GitHub Actions](GitHub-Actions.md)). El resultado aparece en el propio PR como ✅ o ❌ junto a cada check. La gracia es el **momento**: el error se detecta *antes* de fusionar, cuando aún no ha contaminado la rama común.

```
PR #42  funcion-carrito → main
  ✅ CI Backend / test    (build + tests, 47s)
  ❌ CI Frontend / test   (falla el typecheck)   ← esto se arregla ANTES de fusionar
```

Si el check falla, haces más commits en la misma rama y los checks se relanzan solos. Un PR con checks en rojo no debería fusionarse nunca.

**6. Branch protection: convertir la costumbre en regla**

Por defecto, los checks son *informativos*: alguien con prisa aún puede fusionar en rojo, o saltarse el PR y hacer push directo a `main`. La **protección de rama** (Settings → Branches en GitHub) convierte la disciplina en muro: puedes exigir que solo se fusione vía PR, con los checks en verde y/o con aprobación de revisores. A partir de ahí, el gate deja de depender de la buena voluntad.

## Lo que NO hace

- **No es parte de Git** — es una función de la plataforma (GitHub, GitLab...); Git solo conoce ramas y fusiones.
- **No garantiza calidad por sí solo** — es una herramienta para la revisión humana y automática; el criterio sigue siendo del equipo.
- **No fusiona automáticamente** — espera aprobación (y, a menudo, que pasen las comprobaciones).

## Buenas prácticas avanzadas

- **Elige la estrategia de fusión a propósito, no por defecto** — el botón de fusionar suele ofrecer tres modos: *merge commit* (conserva todos tus commits más uno de fusión), *squash and merge* (aplasta todo el PR en un único commit en `main`) y *rebase and merge* (recoloca tus commits uno a uno, sin commit de fusión). No es un detalle estético: define cómo se leerá el historial de `main` para siempre. Los equipos maduros eligen uno y lo fijan en la configuración del repo.
- **Tras un *squash and merge*, la rama está muerta: no la reutilices** — el squash crea un commit **nuevo** en `main` que Git no relaciona con los commits originales de tu rama. Si sigues trabajando sobre ella y abres otro PR, Git intentará meter otra vez los cambios ya fusionados y lloverán conflictos absurdos. Rama fusionada con squash, rama que se borra; lo siguiente arranca de `main` fresco.
- **Sé quien primero revisa tu propio PR** — antes de pedir revisión, recorre tu diff en la interfaz del PR, como lo verá quien revise. Descubrirás archivos colados, código de depuración y renombrados que despistan; y donde algo necesite contexto, déjalo como comentario tuyo en la línea exacta. Un PR auto-revisado y anotado se revisa en la mitad de tiempo y genera la mitad de idas y venidas.
- **Usa PRs en borrador para conversar antes de pulir** — un *draft PR* señala "esto no está listo, pero quiero opinión sobre el enfoque". Recibir un "por ahí no" con el 20 % del trabajo hecho es barato; recibirlo con el 100 % es una semana a la basura. Marcarlo como borrador además evita que salte la petición de revisión formal antes de tiempo.

---

*En resumen: un pull request es el punto de control donde tu trabajo se revisa y se discute antes de entrar en la rama principal — el corazón de la colaboración en equipo.*
