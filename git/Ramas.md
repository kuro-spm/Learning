# Ramas (branches)

## ¿Qué es?

Una rama es una línea de trabajo independiente dentro del proyecto. Te permite hacer cambios sin tocar la versión principal, hasta que estés listo para combinarlos.

## ¿Por qué existe?

Imagina que quieres probar una función nueva en una tienda online, pero no quieres romper la versión que ya usan los clientes. Con una rama, creas una "copia paralela" del proyecto donde experimentas con total libertad. Si sale bien, la combinas; si sale mal, la descartas y la versión principal nunca se enteró.

Esto hace posible que varias personas trabajen a la vez: cada una en su rama, sin pisarse.

> Piensa en un videojuego: la rama es como una partida guardada aparte donde pruebas una estrategia arriesgada. Si funciona, sigues desde ahí; si no, vuelves a tu partida principal intacta.

## ¿Cuándo y para qué se usa?

- Para desarrollar una **función nueva** sin desestabilizar lo que ya funciona.
- Para **arreglar un error urgente** mientras otra persona sigue con su trabajo.
- Para **probar ideas** que quizá descartes.

La rama principal suele llamarse `main`. El resto son ramas temporales que se crean, se combinan y se borran.

## Lo mínimo que necesitas saber

**1. Ver las ramas y en cuál estás**

```bash
git branch
# * main      <- el asterisco marca la rama actual
#   funcion-login
```

**2. Crear una rama y cambiar a ella**

```bash
git switch -c funcion-login
```

`-c` crea la rama y te cambia a ella de un golpe. (En versiones antiguas se usaba `git checkout -b`.)

**3. Cambiar entre ramas existentes**

```bash
git switch main
```

Al cambiar de rama, los archivos de tu carpeta se actualizan para reflejar esa línea de trabajo.

**4. Borrar una rama ya combinada**

```bash
git branch -d funcion-login
```

## Lo que NO hace

- **Crear una rama no copia tu proyecto a otra carpeta** — Git gestiona todo en la misma, cambiando los archivos al vuelo. Es muy ligero.
- **Cambiar de rama no combina nada** — solo te mueve. Combinar es otro paso (ver [Fusión y conflictos](Fusion-y-conflictos.md)).
- **No guarda tu trabajo a medias automáticamente** — confirma o guarda tus cambios antes de cambiar de rama para no perderlos.

---

*En resumen: una rama es un universo paralelo de tu proyecto — experimentas sin miedo, porque la versión principal sigue a salvo hasta que decides combinar.*
