# Remotos (push, pull, fetch)

## ¿Qué es?

Un *remoto* es una copia del repositorio alojada en otro sitio, normalmente un servidor en internet como GitHub o GitLab. Sincronizar con él te permite compartir tu trabajo y recibir el de los demás.

## ¿Por qué existe?

Git es distribuido: cada quien tiene el repo completo en su ordenador. Pero para colaborar (o tener una copia a salvo fuera de tu máquina) hace falta un punto común al que todos suban y del que todos bajen. Ese punto es el remoto.

> Piensa en el remoto como una carpeta compartida en la nube para tu proyecto: cada persona trabaja en su copia local y luego sube y baja cambios del lugar común.

## ¿Cuándo y para qué se usa?

- Para **publicar** tu proyecto y que otros puedan verlo o colaborar.
- Para **trabajar en equipo**: tú subes tus commits, tus compañeros bajan los suyos.
- Para tener una **copia de seguridad** del historial fuera de tu ordenador.

El remoto por defecto suele llamarse `origin`.

## Lo mínimo que necesitas saber

**1. Conectar tu repo local con un remoto**

```bash
git remote add origin https://github.com/usuario/mi-blog.git
```

(Si clonaste el proyecto con `git clone`, esto ya está hecho.)

**2. Subir tus commits (`push`)**

```bash
git push origin main
```

Envía los commits de tu rama `main` al remoto. La primera vez puedes usar `git push -u origin main` para vincularlas y luego basta con `git push`.

**3. Bajar y combinar los cambios de los demás (`pull`)**

```bash
git pull
```

Trae los commits nuevos del remoto y los fusiona en tu rama actual. Hazlo a menudo para no quedarte desactualizado.

**4. Solo mirar, sin combinar (`fetch`)**

```bash
git fetch
```

Descarga los cambios del remoto pero **no** los fusiona. Útil para revisar qué hay de nuevo antes de integrarlo.

## Lo que NO hace

- **`push` no sube lo que no has commiteado** — solo viajan los commits, no los cambios sueltos.
- **`pull` puede provocar conflictos** — si tú y otra persona tocasteis la misma línea (ver [Fusión y conflictos](Fusion-y-conflictos.md)).
- **El remoto no es Git** — es solo un lugar donde vive una copia; toda la potencia sigue estando en tu Git local.

## Buenas prácticas avanzadas

- **`origin/main` no es el servidor: es tu último apunte de cómo estaba** — las ramas tipo `origin/main` son marcadores *locales* que solo se actualizan cuando haces `fetch` o `pull`. Si llevas dos días sin sincronizar, `git status` puede decirte alegremente "up to date with origin/main" mientras el remoto real va veinte commits por delante. Quien entiende esto hace `git fetch` antes de fiarse de cualquier comparación con el remoto.
- **`git pull --rebase` para no sembrar commits de fusión vacíos** — el `pull` normal fusiona lo remoto con lo tuyo, generando esos commits "Merge branch 'main' of..." que ensucian el historial sin aportar nada. Con `--rebase`, tus commits locales se recolocan encima de lo que llegó, y la historia queda recta. Puede hacerse el comportamiento por defecto con `git config --global pull.rebase true`; es una de las primeras cosas que configura la gente con experiencia.
- **Poda las ramas fantasma con `--prune`** — cuando alguien borra una rama en el remoto, tu copia local de `origin/esa-rama` no desaparece: se queda como un fantasma para siempre. `git fetch --prune` (o `git config --global fetch.prune true` para automatizarlo) elimina esas referencias muertas y mantiene tu vista del remoto fiel a la realidad.

---

*En resumen: los remotos son el punto de encuentro de tu proyecto — subes (`push`) lo tuyo y bajas (`pull`) lo de los demás para que todas las copias hablen el mismo idioma.*
