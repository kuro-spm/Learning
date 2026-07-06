# Coverlet

**¿Qué es?** El colector de **cobertura de código** estándar en .NET: mide qué líneas y ramas de tu código de producción se ejecutan durante los tests. El paquete `coverlet.collector` se añade al proyecto de tests y se activa con un flag de `dotnet test`.

---

## ¿Por qué existe?

Los tests pasan en verde, pero ¿cuánto código están ejercitando realmente? Sin medición, la respuesta es una intuición. Coverlet instrumenta los ensamblados durante la ejecución y produce un informe: qué porcentaje de líneas se ejecutó, qué ramas de cada `if` se probaron y — lo más útil — qué zonas no pisa ningún test.

---

## ¿Cuándo y para qué se usa?

- En **CI**, para vigilar la evolución: que un cambio grande no entre con su lógica sin probar.
- En local, para **encontrar huecos**: esa rama del `catch` que nunca se ejecuta, el método de validación que nadie llama.

---

## Lo mínimo que necesitas saber

**1. Recoger cobertura al ejecutar los tests**

```bash
dotnet test --collect:"XPlat Code Coverage"
```

Genera un `coverage.cobertura.xml` en `TestResults/` — un formato estándar que entienden GitHub Actions, Azure DevOps, SonarQube, etc.

**2. Convertirlo en un informe legible**

```bash
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coveragereport"
```

Abre `coveragereport/index.html`: cada archivo con sus líneas en verde (cubiertas) y rojo (no cubiertas).

**3. Umbrales de cobertura: hacer que el build falle por debajo del mínimo**

Un umbral convierte la cobertura de dato informativo en regla de equipo: si baja del mínimo, `dotnet test` falla y el pipeline se pone en rojo. Requiere el paquete `coverlet.msbuild` (el colector `coverlet.collector` no soporta umbrales por sí solo):

```bash
dotnet test /p:CollectCoverage=true /p:Threshold=80 /p:ThresholdType=branch
```

- `Threshold` — porcentaje mínimo exigido.
- `ThresholdType` — sobre qué se mide: `line`, `branch` o `method` (se pueden combinar: `line,branch`).
- `ThresholdStat` — cómo se evalúa: `minimum` (cada proyecto debe cumplirlo, el valor por defecto), `total` (la media global) o `average`.

Alternativa sin cambiar de paquete: dejar que `reportgenerator` o el propio CI (por ejemplo, un check de GitHub Actions) comparen el `coverage.cobertura.xml` contra el mínimo acordado.

---

## Lo que NO hace

- **No mide la calidad de los tests** — una línea "cubierta" solo significa que se ejecutó, no que alguien asertó sobre su resultado. Puedes tener 90% de cobertura con tests que no comprueban nada.
- **No ejecuta nada por sí mismo** — es un colector pasivo; quien ejecuta los tests es xUnit vía `dotnet test`.

---

## Buenas prácticas avanzadas

- **La cobertura de ramas dice más que la de líneas** — un `if` con una sola rama probada cuenta como línea cubierta, pero la mitad de su comportamiento sigue sin test. Cuando revises el informe, mira *branch coverage*.
- **No persigas el 100%** — forzar cobertura total produce tests que ejecutan código sin asertar nada, solo para pintar líneas de verde. Un objetivo razonable con excepciones justificadas (código generado, DTOs) vale más que un número perfecto.
- **Usa la cobertura como detector de huecos, no como nota** — la pregunta útil no es "¿qué porcentaje tengo?" sino "¿qué hay en rojo y me da miedo que esté en rojo?".
- **El umbral funciona como trinquete, no como aspiración** — fija el mínimo justo por debajo de la cobertura real actual y súbelo a medida que mejora. Un umbral muy por encima de la realidad se desactiva el primer viernes con prisas, y ya no vuelve.

---

*En resumen: coverlet te enseña qué parte de tu código los tests ni siquiera pisan — un mapa de huecos, no una medalla.*
