# Deshacer cambios

## ¿Qué es?

Un conjunto de comandos para corregir errores: descartar cambios que no quieres, sacar archivos del área de preparación o anular commits que ya hiciste.

## ¿Por qué existe?

Equivocarse es parte de programar: guardas algo a medias, preparas un archivo que no tocaba o confirmas un cambio que rompe todo. Git guarda historial precisamente para que estos errores no sean permanentes. La clave es elegir la herramienta según **en qué punto** está el error.

> Es como el "deshacer" (Ctrl+Z) de cualquier programa, pero con varios niveles: hay uno para los cambios sin guardar, otro para lo que ya preparaste y otro para lo que ya confirmaste.

## ¿Cuándo y para qué se usa?

Depende de dónde esté el cambio que quieres deshacer:

- Aún **sin preparar** (solo modificado).
- Ya **preparado** (en el área de preparación).
- Ya **confirmado** (en el historial).

## Lo mínimo que necesitas saber

**1. Descartar cambios de un archivo sin guardar**

```bash
git restore index.html
```

Devuelve el archivo a como estaba en el último commit. ⚠️ Pierdes esos cambios: úsalo solo si estás seguro.

**2. Sacar un archivo del área de preparación**

```bash
git restore --staged index.html
```

Lo quita de la "caja" del próximo commit, pero conserva tus cambios.

**3. Anular un commit ya hecho, de forma segura (`revert`)**

```bash
git revert a1b2c3d
```

Crea un **nuevo** commit que deshace lo que hizo aquel. No borra historial, así que es seguro incluso si ya compartiste el commit con el equipo.

**4. Mover el historial hacia atrás (`reset`) — con cuidado**

```bash
git reset --soft HEAD~1   # deshace el último commit, conserva los cambios preparados
git reset --hard HEAD~1   # deshace el último commit Y descarta los cambios
```

`--hard` es destructivo: borra cambios sin posibilidad fácil de recuperarlos. Evítalo en commits que ya hayas subido a un remoto compartido.

## Lo que NO hace

- **`revert` no borra el commit original** — añade uno nuevo que lo neutraliza, dejando rastro de lo ocurrido.
- **`restore` y `reset --hard` no preguntan** — descartan cambios al instante; asegúrate antes.
- **No recupera lo que nunca se confirmó** — Git solo puede devolverte a estados que en algún momento guardaste.

## Buenas prácticas avanzadas

- **`git reflog` es tu red de seguridad definitiva** — registra cada posición por la que ha pasado tu rama, incluso las "borradas" por un `reset --hard` o un rebase fallido. Tras el susto, `git reflog` te enseña el identificador del estado anterior y `git reset --hard HEAD@{1}` te devuelve a él. Los commits huérfanos sobreviven semanas antes de que Git los limpie, así que casi ningún desastre con cosas *ya confirmadas* es definitivo — interiorizarlo es lo que separa el pánico de la rutina.
- **Domina los tres modos de `reset` como un dial** — `--soft` solo mueve la rama (los cambios quedan preparados), el modo por defecto `--mixed` además vacía el área de preparación (los cambios quedan como modificados) y `--hard` también machaca tus archivos. Solo el último destruye trabajo: `git reset HEAD~1` a secas es una operación segura para "descommitear" y recolocar, y usarla sin miedo es señal de dominio.
- **Revertir un commit de fusión tiene trampa doble** — necesita `git revert -m 1 <commit>` para indicar cuál de los dos padres es la línea principal. Y lo sutil: si más adelante intentas volver a fusionar esa misma rama, Git "recordará" que sus commits ya entraron una vez y no traerá nada — hay que revertir el revert o rehacer los commits. Es un clásico que desconcierta incluso a gente con años de Git.

---

*En resumen: deshacer en Git es elegir la herramienta según dónde esté el error — `restore` para lo no confirmado, `revert` para anular un commit sin riesgo y `reset` solo cuando sabes lo que haces.*
