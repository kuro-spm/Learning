# Desarrollo dirigido por especificación

## ¿Qué es?

Es trabajar con un agente en dos tiempos separados: primero se acuerda **qué** hay que construir, por escrito y con detalle suficiente para no dejar huecos; después se implementa contra ese documento. La especificación deja de ser papeleo y pasa a ser el artefacto que se revisa, porque revisar una decisión cuesta minutos y revisar su implementación cuesta horas.

## ¿Por qué existe?

Porque el cuello de botella se ha movido. Cuando escribir 300 líneas de código costaba una tarde, tenía sentido invertir el esfuerzo en escribirlas bien. Ahora esas 300 líneas salen en dos minutos, y el trabajo caro es otro: **decidir qué 300 líneas** y **comprobar que son las correctas**.

El modo natural de trabajar con un agente —pedir algo, ver qué sale, corregir, repetir— funciona bien mientras la tarea sea pequeña. En cuanto toca varios ficheros, el patrón se rompe siempre igual: el agente implementa algo razonable basado en una suposición que no compartes, tú lo descubres a mitad de la revisión, y hay que decidir entre corregir sobre una base equivocada o tirarlo. La especificación previa existe para que esa suposición aparezca cuando todavía es un párrafo.

> Es la misma lógica por la que un equipo revisa el diseño de una API antes de implementarla, no después. Lo único que cambia es la escala: como implementar ahora es rapidísimo, la desproporción entre «arreglar el diseño» y «arreglar la implementación» es mucho mayor que antes, y por tanto el retorno de especificar es mucho más alto.

## ¿Cuándo y para qué se usa?

Cuando la tarea cumple al menos una de estas condiciones:

- **Toca varios ficheros o varias capas** — un endpoint nuevo con su validación, su servicio, su acceso a datos y sus tests.
- **Hay decisiones de diseño con más de una respuesta razonable** — ¿el descuento se calcula al añadir al carrito o al confirmar el pedido?
- **Va a durar más de una sesión** — la especificación es lo que permite retomarla sin reconstruir el contexto de cabeza.
- **La va a revisar otra persona** — el documento es lo que hace revisable un cambio grande generado por IA.

Y **no** se usa para lo pequeño: arreglar un test que falla, renombrar una variable, añadir un campo a un DTO. Escribir una especificación para eso es puro ritual: se tarda más en el documento que en el cambio.

---

## El ciclo completo

Cinco fases. Lo importante es que hay una **puerta de revisión humana** entre las tres primeras y la implementación:

```text
1. EXPLORAR    ¿cómo funciona hoy esto? (solo lectura, nada de escribir)
2. ESPECIFICAR qué debe pasar, con criterios de aceptación
        ↑ revisión humana: aquí se corrigen las decisiones ↑
3. PLANIFICAR  qué ficheros, en qué orden, en cuántos pasos
        ↑ revisión humana: aquí se corrige el enfoque ↑
4. IMPLEMENTAR paso a paso, con un commit por paso
5. VERIFICAR   tests, revisión del diff, criterios de aceptación uno a uno
```

Saltarse la fase 1 es el error más frecuente y el más caro: si el agente no ha leído cómo funciona el sistema actual, la especificación describirá un mundo que no existe.

### Fase 1 — Explorar sin escribir

La instrucción clave es prohibir explícitamente la escritura, porque la tendencia por defecto de un agente es empezar a implementar:

```text
Antes de proponer nada: investiga cómo se calcula hoy el total de un
pedido en este repositorio.

Solo lectura — no modifiques ningún fichero.

Dime:
- Qué clases participan y qué hace cada una.
- Dónde se aplican hoy impuestos o ajustes de precio, si en algún sitio.
- Qué tests cubren el cálculo actual.
- Qué te ha sorprendido o te parece frágil.
```

La última pregunta es la más rentable. «Qué te ha sorprendido» suele destapar lo que un documento de especificación escrito a ciegas nunca habría contemplado: una columna legada, un `if` con un caso especial de hace tres años, un test que en realidad no comprueba nada.

### Fase 2 — La especificación

Una buena especificación es corta y concreta. Este tamaño es el adecuado para una funcionalidad de tamaño medio:

```markdown
# Cupones de descuento en el carrito

## Objetivo
Permitir aplicar un cupón al carrito antes de confirmar el pedido.

## Alcance
Dentro: validar el cupón, aplicar el descuento, mostrarlo desglosado
en el resumen del carrito.
Fuera: crear y administrar cupones (ya existe en el panel de gestión),
cupones acumulables, cupones por producto.

## Comportamiento
- Un cupón se identifica por un código alfanumérico de 6 a 12 caracteres.
- Solo puede haber un cupón activo por carrito. Aplicar uno nuevo
  sustituye al anterior.
- El descuento se calcula sobre el subtotal, antes de gastos de envío.
- El descuento nunca puede dejar el total por debajo de 0.

## Casos de error
| Situación                   | Respuesta                                 |
|-----------------------------|-------------------------------------------|
| Código inexistente          | 400, "Cupón no válido"                    |
| Cupón caducado              | 400, "El cupón ha caducado"               |
| Cupón por debajo del mínimo | 400, "Pedido mínimo de {importe}"         |
| Carrito vacío               | 400, "No hay productos en el carrito"     |

## Criterios de aceptación
- [ ] POST /api/carrito/cupon con un código válido devuelve el carrito
      con `descuento` y `total` recalculados.
- [ ] Aplicar un segundo cupón sustituye al primero, no los suma.
- [ ] Un cupón del 100 % sobre un carrito de 10 € da total 0, nunca negativo.
- [ ] Los cuatro casos de error devuelven 400 con el mensaje indicado.
- [ ] `dotnet test --filter Carrito` en verde.

## Decisiones tomadas
- El descuento se guarda en el carrito, no se recalcula en cada lectura:
  si el cupón caduca entre añadirlo y confirmar, se respeta el precio
  que vio la persona.
```

Tres cosas hacen que esta especificación funcione y que la mayoría no funcionen:

- **El apartado «Fuera» vale tanto como el «Dentro».** Sin él, el agente añade administración de cupones porque «hacía falta». Decir explícitamente qué no se toca es lo que evita la mitad del código no pedido.
- **Los criterios de aceptación están escritos como comprobaciones, no como deseos.** «Debe funcionar bien» no es verificable; «un cupón del 100 % da total 0, nunca negativo» sí. Cada casilla es un test en potencia.
- **Las decisiones quedan registradas con su motivo.** Ese último apartado es lo que evita que en la revisión —o dentro de seis meses— alguien «corrija» algo que era deliberado.

Lo normal es escribirla **con** el agente, no a mano: se le pide un borrador a partir de la exploración y tú corriges. Corregir un borrador es mucho más rápido que redactar desde cero, y además revela las suposiciones del modelo, que es exactamente lo que quieres ver antes de que escriba código.

### Fase 3 — El plan

La especificación dice *qué*; el plan dice *en qué orden* y *en cuántos trozos*:

```text
A partir de la especificación de cupones, propón un plan de implementación:
- Los pasos en orden, cada uno con los ficheros que toca.
- Cada paso debe dejar el proyecto compilando y con los tests en verde,
  para poder hacer commit.
- Marca qué paso es el más arriesgado y por qué.

No implementes todavía.
```

Respuesta típica:

```text
Paso 1 — Modelo y validación (bajo riesgo)
  - Dominio/Cupon.cs (nuevo): entidad y regla de caducidad
  - Dominio/Carrito.cs: campo CuponAplicado + método AplicarCupon
  - Tests/CarritoTests.cs: casos de sustitución y de total nunca negativo

Paso 2 — Acceso a datos (bajo riesgo)
  - Datos/CuponRepository.cs (nuevo): ObtenerPorCodigoAsync
  - Migración: ninguna (la tabla Cupones ya existe)

Paso 3 — Endpoint (riesgo medio: el más arriesgado)
  - Api/CarritoController.cs: POST /api/carrito/cupon
  - Api/Validadores/AplicarCuponRequestValidator.cs (nuevo)
  Riesgo: el controlador hoy devuelve el carrito con un DTO que no tiene
  campo de descuento. Hay que ampliarlo y eso afecta al frontend.

Paso 4 — Tests de integración
  - Tests.Integracion/CuponesTests.cs (nuevo): los cuatro casos de error
```

El paso 3 es el valor entero del ejercicio: **el agente ha detectado una consecuencia que no estaba en la especificación** —el DTO de respuesta afecta al frontend— y la ha señalado antes de implementar. Ese hallazgo, en el modo «pide y reza», habría aparecido cuando el frontend dejara de funcionar.

La regla de que «cada paso deja el proyecto en verde» no es estética: es lo que permite hacer *commit* por paso y, por tanto, deshacer un paso concreto sin tirar los otros tres.

### Fase 4 — Implementar por pasos

Un paso, un *commit*, una comprobación:

```text
Implementa solo el paso 1. Cuando termines, ejecuta
`dotnet test --filter Carrito` y arregla lo que falle. Para y avísame
antes de seguir con el paso 2.
```

El «para y avísame» es lo que mantiene los diffs revisables. Sin él, un agente competente implementa los cuatro pasos de un tirón y te deja 600 líneas para revisar de golpe, que es tanto como no revisar nada.

Y con Git dando puntos de retorno reales:

```bash
git add -A && git commit -m "Añade entidad Cupon y aplicación en el carrito"
# revisas, sigues con el paso 2...
# y si el paso 3 sale mal:
git reset --hard HEAD   # se pierde el paso 3, no los pasos 1 y 2
```

### Fase 5 — Verificar contra los criterios, no contra la sensación

La verificación es recorrer la lista de criterios de aceptación uno a uno. Se puede delegar la comprobación mecánica, pero conviene pedirla de una forma concreta:

```text
Recorre los criterios de aceptación de la especificación uno a uno.
Para cada uno: dime si está cumplido y con qué evidencia concreta
(nombre del test, salida del comando). Si algo no está verificado,
dilo explícitamente en vez de asumir que está bien.
```

Esa última frase importa: pedir **evidencia** en vez de un veredicto reduce mucho los «hecho» optimistas. Un criterio que solo se puede justificar con «lo he implementado» no está verificado.

---

## Los tests como especificación ejecutable

Hay un atajo que funciona muy bien y que conviene conocer: en vez de prosa, escribir la especificación como tests que fallan.

```text
Escribe los tests de la especificación de cupones. Solo los tests —sin
implementación. Deben fallar por falta de funcionalidad, no por errores
de compilación en el propio test.

Cuando los tenga revisados, implementas hasta que pasen.
```

Ventajas sobre la prosa: es imposible de malinterpretar, se verifica sola, y queda en el repositorio como documentación que no se desactualiza. Desventaja: no puede expresar el *por qué* ni las decisiones descartadas.

En la práctica lo mejor es combinarlos: prosa corta para el objetivo, el alcance y las decisiones; tests para el comportamiento y los casos de error. La prosa recoge lo que un test no puede decir; los tests recogen lo que la prosa no puede garantizar.

---

## El anti-patrón: la especificación de cuarenta páginas

Es el error del que se pasa de rosca. Síntomas: un documento enorme, escrito antes de explorar el código, lleno de detalles de implementación («el método se llamará `AplicarCuponAsync` y recibirá un `CancellationToken`») y de secciones genéricas que valdrían para cualquier proyecto.

Por qué falla:

- **Gasta contexto sin aportar señal.** Cuarenta páginas de especificación son decenas de miles de tokens que compiten con el código real por el espacio en la ventana.
- **Especifica lo que no importa y deja lo que sí.** El nombre del método es irrelevante; que un cupón del 100 % no pueda dejar el total en negativo es crítico. Los documentos largos suelen tener mucho de lo primero y poco de lo segundo.
- **Envejece a la primera semana** y entonces confunde más de lo que ayuda, porque nadie sabe qué partes siguen siendo verdad.

La prueba de olor: **si un desarrollador de tu equipo no podría implementarlo a partir de tu documento, le falta información; si tarda más en leerlo que en implementarlo, le sobra.** La especificación de cupones de arriba está en el punto correcto, y cabe en una pantalla.

Y una decisión que hay que tomar explícitamente: **qué pasa con el documento cuando el trabajo termina.** Dos opciones válidas: se borra (era un andamio, su valor era la conversación que provocó) o se convierte en documentación de verdad y alguien se responsabiliza de mantenerla. Lo que no funciona es dejarla en el repositorio sin decidirlo, porque acaba siendo una descripción obsoleta de cómo funcionaba el sistema hace ocho meses.

---

## Buenas prácticas avanzadas

- **Prohíbe la escritura durante la exploración, de forma explícita.** «Solo lectura, no modifiques nada» parece innecesario y no lo es: la tendencia por defecto de un agente capaz es empezar a implementar en cuanto entiende la tarea, y un agente que ya escribió código tiene un sesgo evidente al proponer el diseño —defenderá lo que ya hizo. Separar exploración de implementación es lo que mantiene la especificación honesta.
- **Escribe el apartado de «fuera de alcance» antes que el de dentro.** Es el más incómodo de redactar y el que más código no pedido evita. Los modelos actuales son muy capaces y tienden a resolver el problema *completo* que perciben, no el que pediste: sin un límite escrito, «añade cupones al carrito» acaba incluyendo administración de cupones, historial de uso y un panel de estadísticas.
- **Exige que cada paso del plan deje el proyecto compilando y en verde.** Es lo que convierte Git en una red de seguridad real: puedes descartar el paso que salió mal sin perder los tres anteriores. Un plan cuyos pasos intermedios dejan el proyecto roto solo te da dos estados posibles —antes de todo o después de todo— y eso obliga a revisar 600 líneas de una vez o a tirarlas enteras.
- **Pide evidencia en la verificación, nunca un veredicto.** «¿Está cumplido el criterio 3?» invita a un sí. «¿Con qué test o qué salida de comando se demuestra el criterio 3?» obliga a comprobarlo o a admitir que no se ha comprobado. La diferencia entre las dos preguntas es la diferencia entre una verificación y una formalidad.
- **Deja registradas las decisiones descartadas y por qué.** Es la parte que nadie escribe y la que más valor tiene a medio plazo: sin ella, la siguiente persona —o el siguiente agente, o tú en marzo— «arregla» algo que era deliberado. Dos líneas por decisión bastan.
- **Decide desde el principio si la especificación es andamio o documentación.** Ambas opciones son correctas; no decidirlo no lo es. Las especificaciones huérfanas en el repositorio son peores que no tener ninguna, porque describen con autoridad un sistema que ya no existe y alguien las va a creer.
- **Ajusta el tamaño del proceso a la tarea, y ten el criterio explícito.** Un umbral que funciona: si el cambio toca un solo fichero y lo tienes claro, ve directo. Si toca tres o más, o hay una decisión con dos respuestas razonables, especifica. Aplicar el ciclo completo a todo es la forma más rápida de que el equipo lo abandone por burocrático.

## Recursos didácticos

- **[Best practices for agentic coding (Anthropic)](https://www.anthropic.com/engineering/claude-code-best-practices)** — el flujo explorar → planificar → implementar → verificar explicado por quien construye la herramienta, con las variantes que funcionan en tareas grandes.
- **[Writing an RFC (Squarespace Engineering)](https://engineering.squarespace.com/blog/2019/the-power-of-yes-if)** — cómo se escribe una propuesta técnica que la gente revisa de verdad. El formato es anterior a los LLM y sigue siendo el mejor modelo para una especificación corta y útil.
- **[Architecture Decision Records](https://adr.github.io/)** — un formato de una página para registrar decisiones con su contexto y sus alternativas descartadas. Encaja directamente como apartado «Decisiones tomadas» de tus especificaciones.

---

*En resumen: cuando implementar es barato, lo caro es decidir y comprobar — así que pon la revisión humana donde están las decisiones, no donde está el código.*
