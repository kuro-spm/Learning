# Commits

## ¿Qué es?

Un commit es una "foto" del proyecto en un momento dado, guardada en el historial. Cada commit registra qué cambió, quién lo hizo, cuándo y, gracias a su mensaje, **por qué**.

## ¿Por qué existe?

El commit es la unidad básica del historial de Git. Es el punto al que puedes volver, el cambio que puedes revisar y la pieza que se combina cuando varias personas trabajan juntas. Un buen historial de commits es como un diario del proyecto: permite entender cómo evolucionó sin tener que leerse todo el código.

> Si ya conoces el botón "Guardar" de cualquier programa, un commit es como guardar... pero sin sobrescribir lo anterior. Cada commit se suma a la línea de tiempo, así que siempre puedes volver atrás.

## ¿Cuándo y para qué se usa?

Cada vez que terminas un cambio con sentido propio: arreglar un error, añadir una función, corregir un texto. La buena práctica es hacer commits **pequeños y frecuentes**, cada uno con una sola idea, en lugar de uno gigante al final del día.

## Lo mínimo que necesitas saber

**1. Confirmar lo que tienes preparado**

```bash
git commit -m "Añade el formulario de contacto"
```

El texto tras `-m` es el mensaje: una frase corta que describe el cambio.

**2. Preparar y confirmar en un solo paso**

```bash
git commit -am "Corrige el precio del producto destacado"
```

La `a` prepara automáticamente los archivos *ya conocidos* por Git. Ojo: no incluye archivos nuevos, que siguen necesitando `git add`.

**3. Ver el historial**

```bash
git log --oneline
# a1b2c3d Corrige el precio del producto destacado
# e4f5g6h Añade el formulario de contacto
```

**4. Escribir buenos mensajes**

Un buen mensaje explica el *qué* y, si no es obvio, el *porqué*. Compara:

- ❌ `cambios`
- ✅ `Corrige el cálculo del IVA en el carrito`

El "yo del futuro" (y tus compañeros) te lo agradecerán.

## Lo que NO hace

- **No guarda lo que no has preparado** — solo entra en el commit lo que esté en el área de preparación.
- **No sube el commit a internet** — eso es `git push`, un paso aparte (ver [Remotos](Remotos.md)).
- **No es irreversible** — puedes deshacer o corregir commits (ver [Deshacer cambios](Deshacer-cambios.md)).

---

*En resumen: un commit es una foto con etiqueta de tu proyecto — el ladrillo con el que se construye todo el historial, así que pon en cada uno una sola idea y un mensaje claro.*
