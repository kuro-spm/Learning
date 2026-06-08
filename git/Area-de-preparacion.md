# Área de preparación (staging)

## ¿Qué es?

El área de preparación (*staging area* o *index*) es una zona intermedia donde colocas los cambios que quieres incluir en tu próximo commit, antes de confirmarlos definitivamente.

## ¿Por qué existe?

Casi nunca quieres guardar *todo* lo que has tocado de golpe. Quizá has arreglado un error y, de paso, has dejado notas a medias en otro archivo. El área de preparación te deja elegir **qué cambios entran en cada commit**, para que el historial cuente una historia limpia y ordenada en lugar de mezclar cosas sin relación.

> Imagina que preparas una caja para enviarla por correo: el área de preparación es la caja. Vas metiendo solo lo que quieres enviar en este envío; lo demás se queda fuera hasta el siguiente.

## ¿Cuándo y para qué se usa?

Cada vez que vas a hacer un commit. Primero "preparas" (`add`) los cambios concretos que quieres guardar, y luego los confirmas. Es útil sobre todo cuando has tocado varias cosas y quieres separarlas en commits distintos.

## Lo mínimo que necesitas saber

**1. Ver qué hay preparado y qué no**

```bash
git status
```

Muestra en verde lo que ya está preparado y en rojo lo que está modificado pero todavía no.

**2. Preparar un archivo concreto**

```bash
git add productos.html
```

**3. Preparar todo lo modificado de una vez**

```bash
git add .
```

El punto significa "todos los cambios de esta carpeta hacia abajo". Cómodo, pero revisa antes con `git status` para no incluir algo sin querer.

**4. Sacar algo del área de preparación**

```bash
git restore --staged productos.html
```

Esto lo quita de la "caja" sin deshacer tus cambios: el archivo vuelve a estar solo modificado.

## Lo que NO hace

- **No guarda nada en el historial** — preparar es solo el paso previo; hasta que no haces `commit`, nada queda registrado.
- **No detecta automáticamente lo que quieres** — eres tú quien decide qué se prepara.
- **No incluye archivos nuevos con `git commit` directamente** — un archivo que Git nunca ha visto debe pasar primero por `git add`.

---

*En resumen: el área de preparación es la caja donde eliges, cambio a cambio, qué entra en tu próximo commit — el filtro que mantiene tu historial limpio.*
