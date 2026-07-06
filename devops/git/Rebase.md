# Rebase

## ¿Qué es?

Hacer *rebase* es reescribir el historial de una rama para que sus commits arranquen desde un punto más reciente de otra rama. En la práctica, mueve tus cambios "encima" del trabajo más nuevo, dejando una línea de tiempo recta.

## ¿Por qué existe?

Cuando trabajas en una rama mientras `main` sigue avanzando, hay dos formas de ponerte al día: fusionar (`merge`) o rebasar (`rebase`).

- **Merge** junta las dos líneas con un commit de fusión extra. El historial conserva la realidad de que hubo dos ramas paralelas, pero queda más enredado.
- **Rebase** finge que empezaste tu trabajo *hoy*, sobre el último estado de `main`. El historial queda como una línea recta, mucho más fácil de leer.

> Imagina que escribiste tu capítulo partiendo de la edición 3 de un libro, pero ya va por la edición 5. Hacer rebase es reescribir tu capítulo como si lo hubieras basado en la edición 5 desde el principio.

## ¿Cuándo y para qué se usa?

- Para **actualizar tu rama** con lo último de `main` sin ensuciar el historial con commits de fusión.
- Para **limpiar tus commits** antes de compartirlos: unir varios en uno, reordenarlos o reescribir mensajes (rebase interactivo).

## Lo mínimo que necesitas saber

**1. Reaplicar tu rama sobre lo último de `main`**

```bash
git switch funcion-login
git rebase main
```

Tus commits se vuelven a aplicar uno a uno encima de `main`.

**2. Rebase interactivo: limpiar tus commits**

```bash
git rebase -i HEAD~3   # los últimos 3 commits
```

Se abre un editor donde puedes `pick` (mantener), `squash` (unir con el anterior), `reword` (cambiar el mensaje) o reordenar.

**3. Si hay conflictos, se resuelven y se continúa**

```bash
# resuelves el archivo en conflicto, luego:
git add archivo.txt
git rebase --continue
```

Para abortar y volver a como estabas: `git rebase --abort`.

## Lo que NO hace

- **No debe usarse en commits ya compartidos** — rebase reescribe el historial; si otras personas ya tienen esos commits, les romperás el suyo. Regla de oro: **rebasa solo lo que aún no has subido**.
- **No combina ramas como merge** — sirve para reordenar/mover commits, no para integrar el trabajo de otra rama de forma definitiva.
- **No es reversible "sin red"** — aunque `git reflog` puede salvarte, mejor entender lo que haces antes.

## Buenas prácticas avanzadas

- **`--fixup` + `--autosquash`: correcciones que se colocan solas** — cuando la revisión te pide retocar un commit antiguo de tu rama, haz `git commit --fixup a1b2c3d`: crea un commit marcado como "arreglo de aquel". Al final, `git rebase -i --autosquash main` coloca y funde cada arreglo con su commit original automáticamente. Trabajas rápido durante la revisión y entregas un historial impecable sin ordenar nada a mano.
- **`--onto` para trasplantar una rama entera** — `git rebase --onto main rama-base mi-rama` arranca `mi-rama` de donde creció y la replanta sobre `main`. Es la salida elegante al error de haber creado tu rama encima de otra rama de función en vez de sobre `main`: mueves solo *tus* commits, sin arrastrar los ajenos.
- **Tras rebasar lo ya subido, empuja con `--force-with-lease`** — si rebasas una rama que ya estaba en el remoto (tu propia rama de PR, por ejemplo), el push normal será rechazado y necesitarás forzar. `git push --force-with-lease` fuerza *solo si* el remoto sigue como lo viste por última vez; si un compañero subió algo entre medias, aborta en lugar de borrárselo. El `--force` a secas no comprueba nada: destierra ese hábito.
- **`--update-refs` para ramas apiladas** — si tienes varias ramas encadenadas (la B construida sobre la A), rebasar la de arriba con `git rebase --update-refs` mueve también los punteros de las ramas intermedias, en vez de dejarlas huérfanas apuntando a commits viejos. Antes esto exigía rebasar rama a rama con `--onto`; ahora es un flag.

---

*En resumen: rebase reescribe tu historia para dejarla recta y limpia — potentísimo para ordenar tus propios commits, pero nunca lo apliques sobre lo que ya compartiste.*
