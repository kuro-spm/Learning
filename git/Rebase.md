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

---

*En resumen: rebase reescribe tu historia para dejarla recta y limpia — potentísimo para ordenar tus propios commits, pero nunca lo apliques sobre lo que ya compartiste.*
