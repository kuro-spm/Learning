# Organizaciones de GitHub — Guía de administración

Todo lo que hay que saber para pasar de "tengo repositorios en mi cuenta personal" a "el equipo tiene una organización gobernada": crear la organización, migrar los repositorios sin perder el historial, repartir permisos con equipos, compartir secretos y entornos entre proyectos, imponer reglas de rama de verdad y dejar instaladas las rutinas de seguridad y revisión.

Está escrita para perfiles de desarrollo que ya usan Git y GitHub a diario —commits, ramas, pull requests, algún workflow de Actions— pero que nunca han administrado el lado de arriba: permisos, políticas, planes y facturación. Cada ficha explica qué es la pieza, por qué existe y cómo se configura, con los comandos de la CLI `gh` al lado de cada concepto para poder comprobarlo sobre una organización real.

---

## Orden de lectura recomendado

### 1. Entender el modelo

Antes de tocar nada, saber qué cambia respecto a una cuenta personal y qué te va a permitir tu plan.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Organizaciones de GitHub](Organizaciones-de-GitHub.md) | Qué es una organización y en qué se diferencia de una cuenta personal. El punto de partida: aquí se explica por qué en una cuenta personal no puede haber dos propietarias. |
| 2 | [Planes y facturación](Planes-y-facturacion.md) | Free, Team y Enterprise, y qué desbloquea cada uno. Leerlo antes de diseñar el flujo de trabajo: media configuración de gobierno solo existe en repositorios privados con plan de pago. |

### 2. Ponerla en marcha

El trabajo de una tarde: crear la organización con ajustes sensatos y traerse los repositorios que ya existen.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [Crear una organización](Crear-una-organizacion.md) | El alta y, sobre todo, los siete ajustes que hay que cambiar mientras la organización está vacía y no molesta a nadie. |
| 4 | [Migrar repositorios desde una cuenta personal](Migrar-repos-desde-cuenta-personal.md) | La transferencia paso a paso: qué se conserva, qué hay que rehacer a mano y cuál es el único paso irreversible. |

### 3. Quién puede hacer qué

El núcleo de la administración: el modelo de permisos y la estructura que lo hace mantenible.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 5 | [Roles y permisos](Roles-y-permisos.md) | Los roles de organización y los cinco permisos de repositorio, cómo se combinan y por qué quitar a alguien de un equipo no siempre le quita el acceso. |
| 6 | [Equipos (teams)](Equipos.md) | La pieza que hace escalable todo lo anterior: permisos a grupos, herencia, revisiones automáticas y `CODEOWNERS`. |

### 4. Automatización y reglas

Lo que necesitan los pipelines y lo que convierte el flujo de trabajo documentado en un control real.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 7 | [Secrets, variables y entornos](Secrets-variables-y-entornos.md) | Los tres ámbitos de configuración de Actions, su precedencia, y los entornos con revisores obligatorios antes de desplegar. |
| 8 | [Rulesets y protección de ramas](Rulesets-y-proteccion-de-ramas.md) | Proteger ramas y tags, con rulesets de organización que aplican a todos los repositorios (también a los que se creen mañana). |

### 5. Seguridad y gobierno

Cómo se mantiene todo lo anterior a lo largo del tiempo.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 9 | [Seguridad de la organización](Seguridad-de-la-organizacion.md) | 2FA obligatorio, SSO, políticas de tokens, Dependabot y *secret scanning*, registro de auditoría y qué hacer ante un incidente. |
| 10 | [Gobierno y buenas prácticas](Gobierno-y-buenas-practicas.md) | Convenciones de nombres, el repositorio `.github`, workflows reutilizables, las rutinas de revisión y los antipatrones más frecuentes. |

---

## Índice completo

<details>
<summary>Ver todos los archivos</summary>

**El modelo**
- [Organizaciones de GitHub](Organizaciones-de-GitHub.md)
- [Planes y facturación](Planes-y-facturacion.md)

**Puesta en marcha**
- [Crear una organización](Crear-una-organizacion.md)
- [Migrar repositorios desde una cuenta personal](Migrar-repos-desde-cuenta-personal.md)

**Permisos**
- [Roles y permisos](Roles-y-permisos.md)
- [Equipos (teams)](Equipos.md)

**Automatización y reglas**
- [Secrets, variables y entornos](Secrets-variables-y-entornos.md)
- [Rulesets y protección de ramas](Rulesets-y-proteccion-de-ramas.md)

**Seguridad y gobierno**
- [Seguridad de la organización](Seguridad-de-la-organizacion.md)
- [Gobierno y buenas prácticas](Gobierno-y-buenas-practicas.md)

</details>

---

> Esta guía cubre el lado administrativo. Para el trabajo diario con el código, las colecciones vecinas: [Git](../git/README.md) (ramas, pull requests, GitHub Actions) y [CI/CD](../ci-cd/README.md) (pipelines, secrets de pipeline, estrategias de despliegue y release por tag).
