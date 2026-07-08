# C# y .NET — Guía de fundamentos

Documentación introductoria del lenguaje C# y la plataforma .NET, pensada para quien ya programa (en cualquier lenguaje) y quiere entender las piezas base sobre las que se construye cualquier aplicación .NET, antes de entrar en frameworks concretos como ASP.NET Core.

La guía separa dos planos: los **fundamentos** (la plataforma, el lenguaje, el build y los paquetes) y las **características del lenguaje** (features e idioms de C#/.NET que aparecen una vez superada la sintaxis básica).

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

---

> ¿Quieres ver estas piezas en acción en un backend web? Los atributos, en particular, son el pan de cada día en ASP.NET Core: echa un vistazo a [Primeros pasos con ASP.NET Core](../../desarrollo-web/asp-net-core/README.md). Y si vienes de escritorio, a [De C# WPF a C# para web](../../desarrollo-web/de-wpf-a-web/README.md).
