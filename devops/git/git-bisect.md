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

## Buenas prácticas avanzadas

- **Automatiza la caza entera con `git bisect run`** — si el error se puede detectar con un comando (un test, un script), no respondas a mano: `git bisect run npm test` prueba cada commit solo, interpretando el código de salida (0 = `good`, distinto de 0 = `bad`) hasta señalar al culpable sin intervención humana. Es la diferencia entre "una tarde depurando" y "lo dejo corriendo mientras como"; con un `git stash show` de por medio, hasta se puede bisectar con el test escrito *después* del bug.
- **`git bisect skip` (o salir con 125) para commits imposibles de probar** — en cualquier historial largo hay commits que no compilan o no arrancan por motivos ajenos al bug que buscas. Marcarlos como `bad` falsearía la búsqueda; `git bisect skip` (o, en un script de `run`, terminar con código 125) le dice a Git "este no se puede evaluar, prueba otro cercano".
- **Con `old`/`new` sirve para cualquier cambio, no solo bugs** — `git bisect start --term-old=lento --term-new=rapido` te deja buscar el commit donde algo *cambió*: cuándo mejoró (o empeoró) el rendimiento, cuándo cambió un comportamiento, cuándo creció el tamaño del bundle. Bisect no busca errores: busca transiciones.

---

*En resumen: `git bisect` es la búsqueda binaria aplicada a tu historial — encuentra el commit que rompió algo en un puñado de pasos, por largo que sea el historial.*
