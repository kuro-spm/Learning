# Git — Guía práctica

Tutorial introductorio de **Git**, el sistema de control de versiones más usado del mundo. Pensado para perfiles junior: cada ficha explica qué es una pieza de Git, por qué existe, cuándo se usa y lo mínimo que necesitas saber, con ejemplos genéricos que se entienden sin conocer ningún proyecto concreto.

Empieza por los fundamentos y avanza en orden: cada bloque se apoya en el anterior. Los dos últimos bloques son más avanzados; puedes dejarlos para cuando domines el flujo básico.

---

## Orden de lectura recomendado

### 1. Fundamentos

Qué es Git y cómo dejarlo listo para trabajar.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Git](Git.md) | Qué es el control de versiones y qué resuelve Git. El punto de partida de todo. |
| 2 | [Instalación y configuración](Instalacion-y-configuracion.md) | Dejar Git instalado y firmado con tu identidad antes del primer commit. |

### 2. El flujo básico

El ciclo que repetirás cada día: editar, preparar, confirmar.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [Repositorio](Repositorio.md) | Convertir una carpeta en un proyecto con historial (`init` / `clone`). |
| 4 | [Área de preparación](Area-de-preparacion.md) | Elegir qué cambios entran en cada commit (`add`). |
| 5 | [Commits](Commits.md) | Guardar cambios en el historial con mensajes claros. |
| 6 | [.gitignore](gitignore.md) | Mantener fuera del historial lo que no debe versionarse. |

### 3. Ramas y combinación

Trabajar en paralelo y volver a juntar el trabajo.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 7 | [Ramas](Ramas.md) | Crear líneas de trabajo independientes para no romper lo que funciona. |
| 8 | [git switch](Switch.md) | Moverte entre ramas y crearlas con el comando moderno y seguro. |
| 9 | [Fusión y conflictos](Fusion-y-conflictos.md) | Combinar ramas y resolver los choques cuando los hay. |

### 4. Trabajar con remotos

Compartir tu trabajo y recibir el de los demás.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 10 | [Remotos](Remotos.md) | Sincronizar con un servidor (`push`, `pull`, `fetch`) para colaborar. |

### 5. Deshacer cambios

Corregir errores según en qué punto estén.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 11 | [Deshacer cambios](Deshacer-cambios.md) | `restore`, `revert` y `reset`: la herramienta correcta para cada error. |

### 6. Técnicas avanzadas

Cuando ya dominas el flujo básico y quieres más control sobre el historial.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 12 | [Rebase](Rebase.md) | Reescribir el historial para dejarlo recto y limpio. Alternativa a `merge`. |
| 13 | [Stash](Stash.md) | Apartar trabajo a medias para cambiar de tarea sin hacer un commit. |
| 14 | [Cherry-pick](Cherry-pick.md) | Llevar un commit suelto de una rama a otra. |
| 15 | [Tags y versiones](Tags-y-versiones.md) | Marcar puntos del historial como versiones publicadas (`v1.0.0`). |
| 16 | [Submódulos](Submodulos.md) | Incluir un repositorio dentro de otro, fijado a una versión concreta. |
| 17 | [git bisect](git-bisect.md) | Encontrar automáticamente el commit que introdujo un error. |

### 7. Colaboración y automatización

Cómo se usa Git en equipo y cómo se automatizan las comprobaciones.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 18 | [Git Flow](Git-Flow.md) | Un modelo de ramas con roles fijos para ordenar el trabajo en equipo. |
| 19 | [Pull requests](Pull-requests.md) | Proponer y revisar cambios antes de fusionarlos. El corazón del trabajo en equipo. |
| 20 | [GitHub Actions](GitHub-Actions.md) | Automatizar tests y despliegues con cada cambio (CI/CD). |

---

## Índice completo

<details>
<summary>Ver todos los archivos</summary>

**Fundamentos**
- [Git](Git.md)
- [Instalación y configuración](Instalacion-y-configuracion.md)

**El flujo básico**
- [Repositorio](Repositorio.md)
- [Área de preparación](Area-de-preparacion.md)
- [Commits](Commits.md)
- [.gitignore](gitignore.md)

**Ramas y combinación**
- [Ramas](Ramas.md)
- [git switch](Switch.md)
- [Fusión y conflictos](Fusion-y-conflictos.md)

**Trabajar con remotos**
- [Remotos](Remotos.md)

**Deshacer cambios**
- [Deshacer cambios](Deshacer-cambios.md)

**Técnicas avanzadas**
- [Rebase](Rebase.md)
- [Stash](Stash.md)
- [Cherry-pick](Cherry-pick.md)
- [Tags y versiones](Tags-y-versiones.md)
- [Submódulos](Submodulos.md)
- [git bisect](git-bisect.md)

**Colaboración y automatización**
- [Git Flow](Git-Flow.md)
- [Pull requests](Pull-requests.md)
- [GitHub Actions](GitHub-Actions.md)

</details>

---

> ¿Te quedas con ganas de más? Buenos siguientes pasos: el modelo más ligero *GitHub Flow* / *trunk-based* como alternativa a Git Flow, los *hooks* de Git para automatizar acciones en local, y `git reflog` para recuperar commits que creías perdidos.
