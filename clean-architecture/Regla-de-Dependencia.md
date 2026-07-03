# Regla de Dependencia

## ¿Qué es?

La regla de dependencia es el principio central de Clean Architecture: **las dependencias del código solo pueden apuntar hacia dentro**, hacia las capas más internas. El código de una capa exterior puede conocer al de una capa interior, pero nunca al revés.

## ¿Por qué existe?

Sin una regla clara, las dependencias acaban siendo un plato de espaguetis: la lógica de negocio importa la librería de la base de datos, que a su vez llama a una función de la pantalla, que necesita un dato del negocio... Todo está enredado con todo y nada se puede cambiar de forma aislada.

La regla de dependencia impone una **única dirección**: lo de fuera depende de lo de dentro. El núcleo (las reglas de negocio) no conoce ni a la base de datos, ni a la web, ni a nada del exterior. Así el corazón de tu aplicación queda protegido de los cambios técnicos.

> Si ya conoces el concepto de "capas" en una aplicación, piensa en esto como una norma de tráfico: las flechas de dependencia son calles de sentido único, y todas van hacia el centro.

## ¿Cuándo y para qué se usa?

Se aplica en cualquier proyecto que siga Clean Architecture (u otras arquitecturas en capas como la hexagonal). Es la regla que decides respetar al escribir cada `import`/`using`: antes de que una clase dependa de otra, te preguntas "¿esta dependencia apunta hacia dentro?".

Por ejemplo, en una tienda online: el cálculo del total de un carrito (lógica de negocio, capa interna) no debe importar nada de la pantalla del carrito ni de la tabla SQL de pedidos (capas externas). Si lo hace, has roto la regla.

## Lo mínimo que necesitas saber

**1. Las flechas apuntan hacia dentro**

```text
Infraestructura ──► Adaptadores ──► Casos de uso ──► Dominio
   (lo externo)                                      (lo interno)
```

Cada capa solo puede depender de las que tiene a su derecha (más internas), nunca de las de su izquierda.

**2. Lo interno no sabe que lo externo existe**

```csharp
// ✅ El caso de uso (interno) usa la entidad Cliente (más interno aún)
public class RegistrarCliente
{
    public void Ejecutar(Cliente cliente) { /* ... */ }
}

// ❌ PROHIBIDO: la entidad Cliente importando algo de la web o la BD
public class Cliente
{
    // using Microsoft.AspNetCore.Mvc;  ← esto rompe la regla
}
```

**3. Cuando lo de dentro necesita algo de fuera, se invierte la dependencia**

A veces un caso de uso necesita guardar datos (algo que vive fuera). En lugar de depender de la base de datos, depende de una **interfaz** que él mismo define. La implementación queda fuera. Esto es la *inversión de dependencias*.

```csharp
// El caso de uso depende de un contrato, no de la tecnología
public class RegistrarCliente(IRepositorioClientes repositorio)
{
    public Task Ejecutar(Cliente cliente) => repositorio.Guardar(cliente);
}
```

**4. Los datos cruzan la frontera como objetos simples**

Cuando un dato pasa de una capa a otra, viaja como un objeto sencillo (un DTO), no como una fila de base de datos ni como un modelo de la pantalla. Así ninguna capa se "contagia" de los detalles de otra.

## Lo que NO hace

- **No prohíbe que las capas se comuniquen** — solo fija la *dirección* de las dependencias, no impide el flujo de datos.
- **No elimina la necesidad de interfaces** — al contrario, las interfaces son la herramienta para cumplirla cuando lo interno necesita algo externo.
- **No se comprueba sola** — ningún compilador la valida por defecto; la respetas tú al diseñar, o con reglas de análisis configuradas a propósito.

## Buenas prácticas avanzadas

- **Haz que el compilador la vigile: proyectos separados** — si dominio e infraestructura son carpetas dentro del mismo proyecto, cualquier `using` prohibido compila sin quejarse. Separarlos en proyectos/assemblies distintos, donde el centro no referencia a la infraestructura, convierte la violación en un error de compilación en vez de en un descuido silencioso.
- **Añade tests de arquitectura** — la disciplina humana falla; herramientas como NetArchTest (C#) o ArchUnit (Java) permiten escribir un test que rompe la build si alguien añade una dependencia hacia fuera:

```csharp
var resultado = Types.InAssembly(typeof(Pedido).Assembly)
    .Should().NotHaveDependencyOn("MiTienda.Infraestructura")
    .GetResult();
Assert.True(resultado.IsSuccessful);
```

- **Las violaciones sutiles no llevan `using` de base de datos** — la regla también se rompe cuando *tipos* del exterior se cuelan en las firmas del centro: un repositorio que devuelve `IQueryable` (con el ORM escondido debajo), una `SqlException` que cruza hasta el caso de uso, o un atributo de serialización del framework sobre un DTO interno. Revisa firmas y excepciones, no solo los imports.
- **La regla habla de compilación, no de ejecución** — en tiempo de ejecución el flujo sigue yendo hacia fuera (el caso de uso acaba escribiendo en la base de datos); lo que apunta hacia dentro es el *grafo de dependencias del código*. Tener clara la diferencia evita discusiones estériles: la regla no prohíbe llamar a la base de datos, prohíbe *conocerla*.

---

*En resumen: la regla de dependencia dice que las flechas del código solo van hacia dentro — el núcleo de negocio nunca debe saber nada del mundo exterior.*
