# git switch

## ¿Qué es?

`git switch` es el comando para **cambiar de rama** (y opcionalmente crearla). Es la forma moderna y específica de moverte entre líneas de trabajo.

## ¿Por qué existe?

Durante años, una sola orden (`git checkout`) hacía demasiadas cosas: cambiaba de rama, creaba ramas y también descartaba cambios de archivos. Eso confundía, porque el mismo comando servía para acciones muy distintas y un despiste podía hacerte perder trabajo.

Para arreglarlo, Git dividió esas tareas en dos comandos con nombres claros: `git switch` para moverte entre ramas y `git restore` para tocar archivos. Cada uno hace una cosa y se lee solo.

> Si vienes de `git checkout`, piensa en `git switch` como la "mitad de ramas" de aquel comando: lo mismo, pero sin riesgo de descartar archivos por error.

## ¿Cuándo y para qué se usa?

- Para **moverte a otra rama** y seguir trabajando allí (por ejemplo, pasar de `main` a la rama donde desarrollas el carrito de una tienda online).
- Para **crear una rama nueva** y situarte en ella en un solo paso.
- Para **volver a la rama anterior** rápidamente cuando saltas entre dos tareas.

## Lo mínimo que necesitas saber

**1. Cambiar a una rama existente**

```bash
git switch main
```

Los archivos de tu carpeta se actualizan para reflejar esa rama.

**2. Crear una rama y cambiar a ella (`-c`)**

```bash
git switch -c funcion-login
```

`-c` ("create") crea la rama a partir de donde estás y te mueve a ella. Equivale al antiguo `git checkout -b`.

**3. Volver a la rama en la que estabas antes**

```bash
git switch -
```

El guion significa "la rama anterior", como el "atrás" del navegador. Muy cómodo para alternar entre dos ramas.

**4. Salir a un commit concreto sin rama (modo exploración)**

```bash
git switch --detach a1b2c3d
```

Te deja "mirando" ese commit para inspeccionarlo. Es temporal: para trabajar en serio, vuelve a una rama con `git switch main`.

## Lo que NO hace

- **No combina ramas** — solo te mueve de una a otra. Juntar el trabajo es otro paso (ver [Fusión y conflictos](Fusion-y-conflictos.md)).
- **No descarta cambios de archivos** — esa tarea es de `git restore` (ver [Deshacer cambios](Deshacer-cambios.md)). Por eso `switch` es más seguro que el viejo `checkout`.
- **No guarda tu trabajo a medias** — si tienes cambios sin confirmar que chocan con la otra rama, Git te frenará. Confirma o usa [Stash](Stash.md) antes de cambiar.

---

*En resumen: `git switch` es el comando claro para moverte entre ramas y crearlas — hace una sola cosa, y por eso es difícil equivocarse con él.*
