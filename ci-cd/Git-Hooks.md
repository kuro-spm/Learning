# Git hooks (validación local)

**¿Qué es?** Los *hooks* son scripts que Git ejecuta automáticamente en momentos concretos de tu flujo local — el más usado es `pre-commit`, que corre **justo antes de crear cada commit** y puede abortarlo si algo falla.

---

## ¿Por qué existe?

El CI en la nube valida el código, pero tiene dos costes: **minutos de ejecución** (que se pagan) y **tiempo de espera** (subes el commit, esperas al robot, descubres que un test fallaba... y vuelta a empezar). Muchos de esos fallos son triviales y se habrían detectado en tu máquina en segundos.

El hook `pre-commit` mueve esa primera validación a local: si el build o los tests rápidos fallan, **el commit ni siquiera se crea**. El error se descubre en el momento más barato posible.

> Es el corrector ortográfico del editor frente a esperar a que el corrector de la imprenta te devuelva el libro: mismo error, detectado mil veces antes.

---

## ¿Cuándo y para qué se usa?

- Para que las **ramas personales no gasten CI**: la validación de tu trabajo diario es local y gratis; el pipeline de pago se reserva para los puntos de integración (PRs a la rama común).
- Para chequeos rápidos y frecuentes: lint, formateo, compilación, tests unitarios. En un monorepo, el hook puede ser *path-aware*: validar solo el área que toca el commit.

---

## Lo mínimo que necesitas saber

**1. Un hook es un script en un momento del ciclo**

```sh
#!/bin/sh
# .githooks/pre-commit — si algo falla (exit != 0), el commit se aborta
set -e
echo "▶ lint + tests rápidos…"
npm run lint && npm test
```

**2. Activarlo (una sola vez por clon)**

```bash
# los hooks no viajan con git clone por seguridad: cada persona los activa
git config core.hooksPath .githooks
```

**3. Path-aware en un monorepo: validar solo lo tocado**

```sh
changed=$(git diff --cached --name-only)
if printf '%s\n' "$changed" | grep -q '^backend/'; then
  ( cd backend && dotnet build && dotnet test --filter "Category=Unit" )
fi
```

**4. Saltárselo conscientemente (y con cuidado)**

```bash
git commit --no-verify   # omite el hook; útil en emergencias, peligroso como costumbre
```

---

## Lo que NO hace

- **No es una barrera de seguridad** — cualquiera puede saltárselo con `--no-verify` o no activarlo; la garantía real sigue siendo el gate del CI en el servidor. El hook es cortesía y velocidad, no control.
- **No debe ser lento** — si tarda minutos, la gente lo desactiva. Tests pesados (integración, E2E) van al CI, no al hook.
- **No se instala solo** — vive en el repo, pero cada persona debe activarlo (o automatizarlo con herramientas como Husky en proyectos Node).

---

## Buenas prácticas avanzadas

- **El hook valida tu carpeta de trabajo, no lo que vas a commitear** — sutileza que casi nadie conoce: `npm test` en el hook corre sobre los archivos tal y como están en disco, incluidos cambios *sin* stagear. Puedes commitear código roto que "pasó el hook" porque un cambio no incluido en el commit lo arreglaba en tu máquina. Herramientas como *lint-staged* (que operan solo sobre lo stageado) o un `git stash --keep-index` antes de validar cierran ese agujero.
- **Reparte el coste: `pre-commit` para lo instantáneo, `pre-push` para lo pesado** — lint y formateo aguantan bien en cada commit; una suite de tests de 40 segundos, no (la gente acaba usando `--no-verify` por sistema, y ahí muere el hook). El hook `pre-push` corre solo al subir, así que tolera validaciones más caras sin castigar cada commit local.
- **El hook debe funcionar sin red y sin sorpresas** — nada de `npm install` ni llamadas a servicios externos dentro del script: un hook que falla porque no hay wifi o porque una API no responde entrena al equipo a saltárselo. Todo lo que ejecute debe estar ya en la máquina y ser determinista.

---

*En resumen: el hook pre-commit atrapa los errores en el momento más barato — tu máquina, antes del commit — y deja el CI de pago para lo que de verdad necesita un árbitro central.*
