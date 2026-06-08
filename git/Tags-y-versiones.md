**¿Qué es?** Una *tag* (etiqueta) es un nombre fijo que pones a un commit concreto para marcarlo como importante, típicamente una versión publicada como `v1.0.0`.

---

## ¿Por qué existe?

Los commits tienen identificadores poco memorables (`a1b2c3d`). Cuando publicas la versión 2.0 de una aplicación, quieres poder volver exactamente a ese punto con un nombre claro. Las tags son ese marcador permanente: "esta foto del proyecto es la versión 2.0".

> Si las ramas son líneas de trabajo en movimiento, una tag es un chincheta clavada en un punto exacto del historial que ya no se mueve.

---

## ¿Cuándo y para qué se usa?

Sobre todo para marcar **versiones publicadas** (*releases*). La convención más común es el *versionado semántico* `vMAYOR.MENOR.PARCHE`:

- **MAYOR** (`v2.0.0`): cambios que rompen compatibilidad.
- **MENOR** (`v1.3.0`): funciones nuevas compatibles.
- **PARCHE** (`v1.2.5`): correcciones de errores.

---

## Lo mínimo que necesitas saber

**1. Crear una tag en el commit actual**

```bash
git tag -a v1.0.0 -m "Primera versión estable"
```

`-a` crea una tag "anotada" (con autor y fecha), lo recomendable para releases.

**2. Listar las tags**

```bash
git tag
# v1.0.0
# v1.1.0
```

**3. Subir las tags al remoto**

```bash
git push origin v1.0.0      # una concreta
git push origin --tags      # todas
```

Las tags **no** se suben con un `git push` normal: hay que enviarlas aparte.

**4. Volver al estado de una versión**

```bash
git switch --detach v1.0.0
```

---

## Lo que NO hace

- **No se mueve sola** — a diferencia de una rama, una tag siempre apunta al mismo commit.
- **No se sube automáticamente** — recuerda el `--tags`.
- **No genera el número de versión por ti** — eres tú quien decide si un cambio es mayor, menor o parche.

---

*En resumen: una tag es la chincheta que marca un punto memorable del historial —casi siempre una versión publicada— para poder volver a él con un nombre claro.*
