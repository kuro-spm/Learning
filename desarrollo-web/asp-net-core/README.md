# Primeros pasos con ASP.NET Core — Guía de tecnologías

Introducción a **ASP.NET Core**, el framework web de .NET, pensada para quien ya programa en C#/.NET (consola, librerías, backend) pero **no ha tocado ASP.NET todavía**. Parte de lo que ya sabes de la plataforma y explica qué añade el framework encima, cuáles son los atributos que verás en el día a día de una API, qué estilos de programación web (MVC, Web API, Minimal APIs, Blazor...) puedes elegir y cómo desplegar la app en producción.

No asume conocimiento previo de web con C#: cada ficha explica qué es, por qué existe, cuándo se usa y lo mínimo para no perderse, con ejemplos genéricos (una tienda online, una API de productos, un formulario de registro).

---

## Orden de lectura recomendado

### 1. El salto desde .NET Core

Antes de los detalles, el cambio de mentalidad: de un programa que hace algo y termina a un servidor que atiende peticiones sin parar.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [De .NET Core a ASP.NET Core](De-NET-Core-a-ASP-NET-Core.md) | Qué añade ASP.NET sobre el .NET que ya conoces: servidor, DI, configuración, middleware y routing. La base de todo lo demás. |

### 2. Los atributos más habituales de una API

Los atributos son la forma en que ASP.NET Core recibe instrucciones declarativas. Estos son los que aparecen en casi cualquier controlador. Si el concepto de "atributo" suena a chino, empieza por [Atributos](../../lenguajes/csharp-dotnet/caracteristicas-del-lenguaje/Atributos.md) en la guía del lenguaje.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 2 | [[ApiController]](ApiController.md) | Convierte un controlador en una API REST con validación e inferencia automáticas. El punto de entrada. |
| 3 | [Enrutado y verbos HTTP](Enrutado-y-Verbos.md) | `[Route]`, `[HttpGet]`, `[HttpPost]`... : cómo se asocia cada método a una URL y un verbo. |
| 4 | [Model binding ([From...])](Model-Binding.md) | `[FromBody]`, `[FromQuery]`, `[FromRoute]`... : de dónde sale el valor de cada parámetro. |
| 5 | [Validación con Data Annotations](Validacion-con-DataAnnotations.md) | `[Required]`, `[Range]`, `[EmailAddress]`... : las reglas que deben cumplir los datos de entrada. |
| 6 | [[Authorize] y [AllowAnonymous]](Authorize.md) | Quién puede acceder a cada endpoint: autenticación, roles y políticas. |

### 3. El panorama completo

Una vez visto MVC de cerca, la vista de pájaro: qué otros estilos ofrece el framework y por qué MVC no te ata a ninguna arquitectura concreta.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 7 | [Modelos de programación web](Modelos-de-programacion-web.md) | MVC, Razor Pages, Web API, Minimal APIs y Blazor comparados, y cómo encajan como capa de presentación de una Clean Architecture. |

### 4. Llevarlo a producción

Ya construida la app, el último paso: sacarla de tu máquina y ponerla a atender peticiones reales.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 8 | [Despliegue](Despliegue.md) | `dotnet publish`, ejecución del artefacto, configuración por entorno, contenedores y Kestrel tras un reverse proxy. |

---

> Los atributos como mecanismo del lenguaje (crear los tuyos, `[AttributeUsage]`, reflection) están en [Atributos](../../lenguajes/csharp-dotnet/caracteristicas-del-lenguaje/Atributos.md), dentro de la guía de [C# y .NET](../../lenguajes/csharp-dotnet/README.md). Y si vienes de escritorio, [De C# WPF a C# para web](../de-wpf-a-web/README.md) cuenta el mismo salto desde la óptica de WPF.
