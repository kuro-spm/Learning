**¿Qué es?** `git stash` guarda temporalmente tus cambios sin confirmar en una "estantería" aparte y te deja la carpeta limpia, para que puedas recuperarlos más tarde.

---

## ¿Por qué existe?

A veces estás a medias de algo y surge una urgencia: tienes que cambiar de rama para arreglar un error, pero Git no te deja porque tienes cambios sin guardar, y todavía no quieres hacer un commit a medias. El *stash* es la solución: aparta lo que tienes, te deja trabajar en otra cosa y luego lo devuelve.

> Es como dejar tu trabajo a medias en un cajón para despejar la mesa, atender otra cosa, y luego sacarlo y seguir donde lo dejaste.

---

## ¿Cuándo y para qué se usa?

Cuando necesitas cambiar de contexto rápidamente (cambiar de rama, bajar cambios del remoto) y tienes trabajo sin terminar que aún no merece un commit.

---

## Lo mínimo que necesitas saber

**1. Guardar los cambios y limpiar la carpeta**

```bash
git stash
```

Tus archivos vuelven al último commit; los cambios quedan guardados aparte.

**2. Recuperar lo guardado**

```bash
git stash pop
```

Devuelve los cambios a la carpeta y los borra de la estantería.

**3. Ver qué tienes guardado**

```bash
git stash list
# stash@{0}: WIP on main: a1b2c3d Añade login
```

**4. Guardar con una etiqueta para acordarte**

```bash
git stash push -m "Formulario a medias"
```

---

## Lo que NO hace

- **No es un sustituto de los commits** — el stash es temporal y local; para guardar trabajo de verdad, haz commit.
- **No se sube al remoto** — vive solo en tu ordenador.
- **No avisa si lo acumulas** — es fácil olvidar stashes viejos; revisa la lista de vez en cuando.

---

## Buenas prácticas avanzadas

- **Los archivos nuevos NO se guardan por defecto** — `git stash` solo aparta cambios de archivos que Git ya conoce; los recién creados se quedan en la carpeta, y es fácil creer que "está todo a salvo" cuando no lo está. `git stash -u` incluye también los archivos sin rastrear; hazlo tu forma habitual de stashear.
- **Si `pop` provoca conflicto, el stash NO se borra** — al aplicar los cambios sobre una base distinta puede haber conflicto; en ese caso `pop` aplica lo que puede pero conserva la entrada en la lista. Resuelto el conflicto, elimínala tú con `git stash drop` — o acabarás aplicando el mismo stash dos veces semanas después sin entender de dónde salen esos cambios.
- **Un stash antiguo se recupera mejor con `git stash branch`** — aplicar un stash de hace dos semanas sobre el código actual suele acabar en conflictos. `git stash branch arreglo-formulario` crea una rama en el commit donde hiciste el stash y lo aplica allí, donde encaja limpio; luego integras esa rama como cualquier otra.

---

*En resumen: `git stash` es el cajón donde guardas trabajo a medias para despejar la mesa un momento y retomarlo después.*
