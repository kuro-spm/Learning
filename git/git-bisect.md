**¿Qué es?** `git bisect` es una herramienta que encuentra automáticamente el commit que introdujo un error, usando una búsqueda binaria por el historial.

---

## ¿Por qué existe?

Imagina que algo funcionaba la semana pasada y hoy está roto, pero entre medias hay 200 commits. Revisarlos uno a uno sería eterno. `git bisect` aplica el truco de "adivina el número": prueba el commit de la mitad, le dices si está bien o mal, y descarta media historia de golpe. En unos pocos pasos te señala el commit culpable exacto, aunque haya miles.

> Es como buscar una palabra en un diccionario: no empiezas por la primera página, abres por la mitad y según la letra saltas a una mitad u otra. En 10 saltos cubres 1.000 páginas.

---

## ¿Cuándo y para qué se usa?

Cuando sabes que algo se rompió en algún punto entre una versión que funcionaba y otra que no, pero no sabes en cuál. Especialmente útil con historiales largos.

---

## Lo mínimo que necesitas saber

**1. Iniciar la búsqueda**

```bash
git bisect start
git bisect bad                 # el commit actual está roto
git bisect good v1.0.0         # esta versión antigua funcionaba
```

**2. Git te lleva al commit del medio; tú lo pruebas y respondes**

```bash
# pruebas si el error sigue ahí, y según el resultado:
git bisect good   # este commit está bien
# o
git bisect bad    # este commit ya tiene el error
```

Git repite, reduciendo el rango a la mitad cada vez, hasta señalar el commit culpable.

**3. Terminar y volver a la normalidad**

```bash
git bisect reset
```

---

## Lo que NO hace

- **No arregla el error** — solo te dice *qué commit* lo introdujo; corregirlo es cosa tuya.
- **No sabe si el código funciona** — eres tú (o un script con `git bisect run`) quien decide si cada commit es `good` o `bad`.
- **No sirve si no tienes un punto bueno conocido** — necesitas un commit "que funcionaba" para acotar la búsqueda.

---

*En resumen: `git bisect` es la búsqueda binaria aplicada a tu historial — encuentra el commit que rompió algo en un puñado de pasos, por largo que sea el historial.*
