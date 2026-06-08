# Backend — Guía de tecnologías

Documentación introductoria del stack backend de **EcoWaveProjectManagement** para desarrolladoras con experiencia en C#/.NET que quieren entender la arquitectura y las librerías concretas que usa el proyecto.

---

## Orden de lectura recomendado

### 1. Lenguaje y Runtime

Aunque ya conoces C#, estos archivos te ubican en las versiones y características específicas del proyecto.

| # | Archivo | Por qué leerlo primero |
|---|---|---|
| 1 | [C#](CSharp.md) | Características modernas de C# usadas en el proyecto (records, pattern matching, nullable). |
| 2 | [.NET 10](DotNET.md) | La plataforma de ejecución y las novedades relevantes de esta versión. |

### 2. Framework Web

El núcleo que expone la API REST al frontend.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [ASP.NET Core Web API](ASPNET-Core-Web-API.md) | Cómo se definen los endpoints, el pipeline HTTP y el enrutamiento. |
| 4 | [MSBuild](MSBuild.md) | Cómo se compila y estructura el proyecto. Útil para entender los .csproj. |

### 3. Arquitectura

Antes de tocar código de negocio, entiende cómo está organizado.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 5 | [Clean Architecture](Clean-Architecture.md) | El patrón general que da estructura a todo el backend. Leer primero. |
| 6 | [Module-based Organization](Module-based-Organization.md) | Cómo se separa el código en módulos (Projectes, Imatges, Dissenys). |
| 7 | [Layer Pattern](Layer-Pattern.md) | Las capas dentro de cada módulo: Application, Infrastructure, Domain. |
| 8 | [Dependency Injection](Dependency-Injection.md) | Cómo .NET inyecta dependencias y cómo se registran los servicios del proyecto. |

### 4. Acceso a Datos

Cómo el backend habla con la base de datos.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 9 | [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md) | El driver de bajo nivel. Leer antes que Dapper. |
| 10 | [Dapper](Dapper.md) | El micro-ORM que mapea resultados SQL a objetos C#. |

### 5. Testing

Cómo se prueban los componentes del backend.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 11 | [xUnit](xUnit.md) | El framework de tests. Punto de entrada obligatorio. |
| 12 | [Microsoft.NET.Test.Sdk](Microsoft-NET-Test-Sdk.md) | El SDK que permite que dotnet test descubra los tests. |
| 13 | [NSubstitute](NSubstitute.md) | Cómo crear mocks de dependencias en los tests unitarios. |
| 14 | [Shouldly](Shouldly.md) | Aserciones más legibles que Assert estándar. |
| 15 | [coverlet.collector](coverlet.md) | Cómo medir la cobertura de código al ejecutar los tests. |
| 16 | [xunit.runner.visualstudio](xunit-runner-visualstudio.md) | Adaptador para ejecutar tests desde el IDE o la línea de comandos. |

---

## Índice completo por categoría

<details>
<summary>Ver todos los archivos</summary>

**Lenguaje y Runtime**
- [C#](CSharp.md)
- [.NET 10](DotNET.md)

**Framework Web**
- [ASP.NET Core Web API](ASPNET-Core-Web-API.md)
- [MSBuild](MSBuild.md)

**Arquitectura**
- [Clean Architecture](Clean-Architecture.md)
- [Module-based Organization](Module-based-Organization.md)
- [Layer Pattern](Layer-Pattern.md)
- [Dependency Injection](Dependency-Injection.md)

**Acceso a Datos**
- [Microsoft.Data.SqlClient](Microsoft-Data-SqlClient.md)
- [Dapper](Dapper.md)

**Testing**
- [xUnit](xUnit.md)
- [xunit.runner.visualstudio](xunit-runner-visualstudio.md)
- [NSubstitute](NSubstitute.md)
- [Shouldly](Shouldly.md)
- [coverlet.collector](coverlet.md)
- [Microsoft.NET.Test.Sdk](Microsoft-NET-Test-Sdk.md)

</details>

---

> Para entender cómo el backend se comunica con el frontend, ve a [/integracion](../integracion/README.md).
