# Monorepo y path-filters

**¿Qué es?** Un *monorepo* es un único repositorio Git que contiene varias partes de un producto (por ejemplo `frontend/` y `backend/` como carpetas). Los *path-filters* son la técnica de CI que hace que cada pipeline se ejecute **solo cuando cambian los archivos de su carpeta**.

---

## ¿Por qué existe?

En un monorepo, cualquier push toca "el repositorio", y sin más información el CI reaccionaría ejecutándolo **todo**: compilar el backend porque alguien corrigió un texto del frontend, o pasar los tests de la web porque se tocó una consulta SQL. Eso desperdicia minutos de CI (que cuestan dinero) y hace las validaciones lentas.

> Piensa en un edificio de dos pisos con un timbre único: sin filtros, cada visita despierta a todos los vecinos. Los path-filters son ponerle a cada piso su propio timbre.

---

## ¿Cuándo y para qué se usa?

Siempre que un repositorio contenga más de una unidad desplegable o compilable por separado. El caso típico: una tienda online con la API en `backend/` y la web en `frontend/`. Cada área tiene su propio workflow, y cada workflow declara qué rutas le importan.

---

## Lo mínimo que necesitas saber

**1. Declarar los paths en el disparador (GitHub Actions)**

```yaml
# .github/workflows/backend-ci.yml
on:
  pull_request:
    branches: [develop]
    paths: ['backend/**']   # solo se ejecuta si el PR tocó backend/
```

**2. Compilar desde la subcarpeta, no desde la raíz**

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend   # todos los comandos corren dentro de backend/
```

**3. El mismo criterio aplica al construir imágenes Docker**

```yaml
- uses: docker/build-push-action@v6
  with:
    context: ./backend    # el Dockerfile y su contexto viven en la subcarpeta
```

---

## Lo que NO hace

- **No detecta dependencias entre áreas** — si el frontend consume un contrato del backend y este cambia, el filtro no lo sabe: eso se gestiona con disciplina de contratos o herramientas de grafo de dependencias (Nx, Turborepo).
- **No sustituye a los tests** — solo decide *cuándo* ejecutarlos.
- **No aísla permisos** — todo el monorepo comparte un único juego de secrets y de reglas de protección de ramas.

---

## Buenas prácticas avanzadas

- **Path-filters y *required checks* se llevan mal** — si marcas "backend-ci" como check obligatorio de la PR y el filtro de paths hace que ese workflow ni se ejecute (porque solo tocaste `frontend/`), la PR se queda esperando un check que nunca llegará. La solución en GitHub Actions es conocida por pocos: un segundo workflow con el **mismo nombre de job** y los paths invertidos (`paths-ignore`) que termina en verde al instante, o mover el filtrado dentro del workflow con una action como `dorny/paths-filter`.
- **Incluye en los paths el propio workflow y lo compartido** — un filtro `paths: ['backend/**']` tiene dos agujeros: cambiar `.github/workflows/backend-ci.yml` no dispara el pipeline que acabas de modificar, y tocar una carpeta común (`shared/`, contratos, configuración raíz) tampoco. Añade siempre esas rutas al filtro; si no, el CI valida todo *excepto* los cambios más peligrosos.

---

*En resumen: en un monorepo, los path-filters le dan a cada carpeta su propio timbre — solo se despierta el pipeline del área que realmente cambió.*
