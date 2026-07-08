# NuGet

## ¿Qué es?

NuGet es el gestor de paquetes oficial de .NET: el sistema que descarga, versiona y referencia librerías (de terceros o propias) para que un proyecto pueda usarlas sin copiar código a mano.

## ¿Por qué existe?

Cualquier aplicación mínimamente grande depende de código que no ha escrito nadie del equipo: serialización JSON, clientes HTTP, ORMs, librerías de testing. Sin un gestor de paquetes, cada dependencia habría que descargarla y copiar sus ficheros a mano, resolviendo a pulso qué pasa cuando dos librerías necesitan versiones distintas de una tercera. NuGet centraliza la distribución (a través de nuget.org o un feed privado) y la resolución de versiones, integrado directamente en el `.csproj` y resuelto en cada compilación por `dotnet restore`.

> Si conoces npm de JavaScript: NuGet es el mismo concepto — un registro público de paquetes y un fichero de proyecto (`.csproj` en vez de `package.json`) que declara qué versión de cada uno necesitas.

## ¿Cuándo y para qué se usa?

Cada vez que un proyecto .NET necesita una librería externa: un ORM como Dapper, un framework de testing como xUnit, una librería de logging como Serilog. También al publicar código propio reutilizable entre varios proyectos como un paquete interno.

## Lo mínimo que necesitas saber

**1. Las dependencias se declaran en el `.csproj`**

```xml
<ItemGroup>
  <PackageReference Include="Dapper" Version="2.1.35" />
  <PackageReference Include="Serilog.AspNetCore" Version="8.0.3" />
</ItemGroup>
```

**2. Añadir un paquete desde la CLI**

```bash
dotnet add package FluentValidation
```

**3. Versionado semántico (SemVer): `MAYOR.MENOR.PARCHE`**

```text
2.1.35
│ │  └─ parche: corrección de errores, siempre compatible
│ └──── menor: funcionalidad nueva, compatible hacia atrás
└────── mayor: puede romper compatibilidad
```

Un paquete que respeta SemVer promete que subir de versión de parche o menor nunca rompe tu código; subir de versión mayor sí puede hacerlo.

**4. Fijar una versión exacta (*pinning*) frente a aceptar un rango**

```xml
<!-- Versión exacta: solo cambia si tú lo decides explícitamente -->
<PackageReference Include="MediatR" Version="12.4.1" />

<!-- Rango: acepta cualquier versión dentro del intervalo -->
<PackageReference Include="MediatR" Version="[12.0.0,13.0.0)" />
```

**5. Gestión centralizada de versiones en soluciones con muchos proyectos**

Con *Central Package Management*, un único fichero fija la versión de cada paquete para toda la solución, evitando que dos proyectos usen sin querer versiones distintas del mismo paquete.

```xml
<!-- Directory.Packages.props, en la raíz de la solución -->
<ItemGroup>
  <PackageVersion Include="MediatR" Version="12.4.1" />
  <PackageVersion Include="FluentAssertions" Version="7.1.0" />
</ItemGroup>
```

## Lo que NO hace

- **No garantiza compatibilidad solo por seguir SemVer** — es una convención que el autor del paquete puede no respetar al pie de la letra; los tests del propio proyecto siguen siendo la única prueba real tras actualizar.
- **No avisa de cambios en la licencia de un paquete** — instalar o actualizar una dependencia no comprueba si sus condiciones de uso han cambiado; eso hay que revisarlo aparte.
- **No sustituye a MSBuild** — NuGet resuelve *qué* versión de cada paquete usar; MSBuild es quien compila el proyecto con esas referencias ya resueltas (ver [MSBuild](MSBuild.md)).

## Buenas prácticas avanzadas

- **Antes de subir una versión mayor, revisa el modelo de licencia, no solo el changelog técnico** — varias librerías muy usadas en el ecosistema .NET (MediatR a partir de la v13, FluentAssertions a partir de la v8, entre otras) han pasado de licencia abierta a un modelo comercial de pago en una versión mayor concreta. Una actualización automática puede subirte sin darte cuenta a una versión que ya no es gratuita para uso comercial; fijar la versión (*pinning*) y revisar manualmente antes de subir de mayor es la forma de evitarlo.
- **Ten decidido el plan B antes de que haga falta** — si un paquete clave cambia de licencia, las opciones suelen ser pagarla, quedarse fijado en la última versión libre de forma indefinida (asumiendo que no llegarán más parches de seguridad), o migrar a una alternativa con licencia abierta equivalente. Es mejor decidirlo con calma que en caliente, el día que toca actualizar por una vulnerabilidad.
- **Vigila las versiones fijadas con `dotnet list package --vulnerable` y `--deprecated`** — quedarte fijado en una versión antigua por precaución de licencia tiene un coste: dejar de recibir parches de seguridad. Conviene comprobar explícitamente, de vez en cuando, si la versión fijada acumula vulnerabilidades conocidas.
- **Usa un *lock file* (`packages.lock.json`) para instalaciones reproducibles** — sin él, dos restauraciones en momentos distintos pueden resolver versiones transitivas ligeramente distintas dentro de los rangos permitidos; el lock file fija exactamente lo que se instaló la última vez, igual que un `package-lock.json` de npm.

---

*En resumen: NuGet resuelve qué versión de cada dependencia usa tu proyecto — fijar versiones explícitamente y vigilar los cambios de licencia en paquetes clave evita que una actualización rutinaria te cambie las condiciones de uso sin avisar.*
