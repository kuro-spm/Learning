# coverlet.collector

## ¿Qué es?

`coverlet.collector` es una librería de .NET que recolecta datos de **cobertura de código** mientras se ejecutan los tests, integrándose directamente con el runner de `dotnet test` sin configuración adicional compleja.

## ¿Por qué existe?

Antes de Coverlet, medir qué porcentaje de tu código ejecutaban los tests requería herramientas de terceros de pago (como NCover) o configuraciones complicadas. Coverlet nació como alternativa open source y multiplataforma.

Si conoces **Visual Studio Enterprise** y su cobertura integrada, Coverlet hace lo mismo pero funciona en cualquier entorno (CI/CD, Linux, Mac) y es gratuito. La variante `coverlet.collector` es la modalidad recomendada actualmente porque se conecta como un *data collector* estándar de VSTest, lo que la hace compatible con la mayoría de herramientas de reporte.

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, `coverlet.collector` está referenciado en los proyectos de test de los módulos (`Projectes`, `Imatges`, `Dissenys`). Se activa automáticamente al ejecutar los tests desde la CLI o el pipeline de CI. Genera los datos de cobertura que luego pueden visualizarse en herramientas como Codecov o el propio Visual Studio.

Se declara como dependencia en el `.csproj` del proyecto de tests:

```xml
<PackageReference Include="coverlet.collector" Version="6.0.4">
  <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  <PrivateAssets>all</PrivateAssets>
</PackageReference>
```

## Lo mínimo que necesitas saber

**1. Se activa con un flag en `dotnet test`**

```bash
dotnet test --collect:"XPlat Code Coverage"
```

**2. Genera un archivo `coverage.cobertura.xml`**

El resultado se guarda en `TestResults/` dentro del proyecto de tests. Este XML es el formato estándar que consumen la mayoría de herramientas de reporte.

**3. Puedes configurar qué excluir**

```xml
<!-- En el .csproj del proyecto de tests -->
<CoverletOutputFormat>cobertura</CoverletOutputFormat>
<ExcludeByAttribute>GeneratedCodeAttribute</ExcludeByAttribute>
```

**4. No necesitas modificar el código bajo test**

Coverlet instrumenta los ensamblados en tiempo de ejecución. Tus clases de `Projectes` o `Dissenys` no necesitan ningún atributo especial.

**5. Integración con `ReportGenerator`**

Para ver un reporte HTML legible:

```bash
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"coverage.cobertura.xml" -targetdir:"coveragereport"
```

## Lo que NO hace

- No decide si tu cobertura es "suficiente" — eso lo configuras tú con umbrales en el pipeline.
- No reemplaza los tests: solo mide qué código ejecutan, no si los tests son correctos.
- No genera tests automáticamente.
- No es un framework de testing (no compite con xUnit ni NUnit).

---

*En resumen: `coverlet.collector` es la pieza silenciosa que, sin tocar tu código de producción, te dice exactamente qué partes de EcoWaveProjectManagement están cubiertas por tests y cuáles no, con solo añadir un flag al comando `dotnet test`.*
