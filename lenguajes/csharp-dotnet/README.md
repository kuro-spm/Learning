# C# y .NET — Guía de fundamentos

Documentación introductoria del lenguaje C# y la plataforma .NET, pensada para quien ya programa (en cualquier lenguaje) y quiere entender las piezas base sobre las que se construye cualquier aplicación .NET, antes de entrar en frameworks concretos como ASP.NET Core.

La guía separa tres planos: los **fundamentos** (la plataforma, el lenguaje, el build y los paquetes), las **características del lenguaje** (features e idioms de C#/.NET que aparecen una vez superada la sintaxis básica) y la **configuración** (cómo una aplicación .NET consume sus ajustes de forma tipada, con el Options pattern).

---

## Orden de lectura recomendado

### 1. Fundamentos

La plataforma, el lenguaje y las herramientas que convierten tu código en un programa que se ejecuta.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [.NET](fundamentos/DotNET.md) | El runtime y la plataforma de ejecución. La base sobre la que corre todo lo demás. |
| 2 | [C#](fundamentos/CSharp.md) | El lenguaje: tipos, sintaxis moderna y cómo se apoya en el runtime de .NET. |
| 3 | [MSBuild](fundamentos/MSBuild.md) | El sistema de build que convierte tu código C# en un ejecutable. |
| 4 | [NuGet](fundamentos/NuGet.md) | El gestor de paquetes: cómo se referencian, versionan y fijan las dependencias de un proyecto. |

### 2. Características del lenguaje

Features del lenguaje y patrones idiomáticos de .NET que vas encontrando al escribir código real, más allá de la sintaxis básica.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 5 | [CancellationToken](caracteristicas-del-lenguaje/CancellationToken.md) | Cómo se propaga de extremo a extremo la petición de cancelar una operación asíncrona. |
| 6 | [Atributos](caracteristicas-del-lenguaje/Atributos.md) | Metadatos declarativos (`[Algo]`) que el framework lee por reflection para decidir cómo tratar tu código. |
| 7 | [Atributos personalizados](caracteristicas-del-lenguaje/Atributos-Personalizados.md) | Crear los tuyos y el porqué de la restricción de constantes de compilación: cómo se graban en los metadatos del ensamblado. |

### 3. Configuración de la aplicación

Cómo una aplicación .NET (de consola, worker o web) lee sus ajustes externos y los expone al código de forma tipada. No es exclusivo de ningún framework: vive en el *host* de cualquier app .NET.

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 8 | [Microsoft.Extensions.Options](Microsoft-Extensions-Options.md) | El Options pattern: bindear una sección de configuración a una clase POCO, consumirla con `IOptions`/`IOptionsSnapshot`/`IOptionsMonitor` y validarla al arrancar. |

---

> ¿Quieres ver estas piezas en acción en un backend web? Los atributos, en particular, son el pan de cada día en ASP.NET Core: echa un vistazo a [Primeros pasos con ASP.NET Core](../../desarrollo-web/asp-net-core/README.md). Y si vienes de escritorio, a [De C# WPF a C# para web](../../desarrollo-web/de-wpf-a-web/README.md).
