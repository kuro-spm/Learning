**¿Qué es?** `git cherry-pick` copia un commit concreto de una rama y lo aplica en otra, sin traer el resto de los cambios de esa rama.

---

## ¿Por qué existe?

A veces solo quieres *un* cambio de otra rama, no todo su trabajo. Por ejemplo, alguien arregló un error en una rama de desarrollo y necesitas ese arreglo —y solo ese— en la rama de producción ya mismo. Fusionar toda la rama traería cosas a medias; cherry-pick toma justo el commit que te interesa.

> Como su nombre indica (*recoger cerezas*), eliges un commit concreto del árbol y te lo llevas, dejando el resto.

---

## ¿Cuándo y para qué se usa?

- Llevar un **arreglo urgente** de una rama a otra sin esperar a fusionar todo.
- Recuperar un **commit suelto** que aplicaste en la rama equivocada.

---

## Lo mínimo que necesitas saber

**1. Localizar el commit que quieres**

```bash
git log --oneline
# f9e8d7c Corrige error de cálculo del total
```

**2. Aplicarlo en tu rama actual**

```bash
git switch produccion
git cherry-pick f9e8d7c
```

Se crea en `produccion` una copia de ese commit.

**3. Si hay conflicto, se resuelve y se continúa**

```bash
git add archivo.txt
git cherry-pick --continue
```

---

## Lo que NO hace

- **No mueve el commit** — lo *copia*; el original sigue en su rama.
- **No trae las dependencias del commit** — si ese cambio se apoyaba en commits anteriores que no están en tu rama, puede fallar o dejar el código a medias.
- **No sustituye a merge ni a rebase** — es para casos puntuales de uno o pocos commits.

---

## Buenas prácticas avanzadas

- **Usa `-x` para dejar rastro del origen** — `git cherry-pick -x f9e8d7c` añade al mensaje una línea `(cherry picked from commit f9e8d7c)`. Meses después, cuando alguien pregunte "¿este arreglo de producción está ya en la rama de desarrollo?", esa línea es la única pista fiable, porque la copia tiene un hash distinto y Git no las relaciona por sí solo.
- **La copia es un commit "gemelo", no el mismo: cuidado al volver a fusionar** — como el original y la copia tienen hashes distintos, si más adelante fusionas la rama de origen completa, Git aplicará dos veces el mismo cambio; a veces lo detecta en silencio y a veces estalla en conflictos confusos. Por eso los equipos expertos usan cherry-pick en direcciones que nunca se re-fusionan (de desarrollo hacia una rama de versión, por ejemplo), no entre ramas que volverán a encontrarse.
- **Para copiar un commit de fusión hace falta `-m 1`** — un merge tiene dos padres y Git no sabe respecto a cuál calcular el cambio; `git cherry-pick -m 1 <commit>` le dice "respecto al primer padre" (la rama que recibió la fusión), que es lo que quieres casi siempre. Sin ese flag, el comando simplemente se niega.

---

*En resumen: cherry-pick es las pinzas con las que tomas un commit suelto de otra rama y lo aplicas en la tuya, sin arrastrar nada más.*
