# Pruebas seguras sobre un Odoo en producción

## ¿Qué es?

Es el conjunto de buenas prácticas para probar cambios en un Odoo que está en uso real —el que la empresa utiliza cada día— **sin arriesgar los datos ni interrumpir el trabajo**. La regla de oro es sencilla: lo que se pueda probar fuera de producción, se prueba fuera; lo que se haga en producción, se hace con red de seguridad.

## ¿Por qué existe?

Producción es el único entorno con datos reales: clientes, facturas, contabilidad, stock. Un error ahí no es un fallo de programación cualquiera: puede enviar correos equivocados a clientes reales, descuadrar la contabilidad o dejar a toda la empresa sin poder trabajar.

Estas prácticas existen para que probar deje de ser una ruleta rusa. No eliminan el riesgo del todo, pero lo reducen a un nivel controlado y siempre reversible.

> Es la diferencia entre un cirujano que opera con quirófano estéril, pruebas previas y un plan de emergencia, y alguien que improvisa con un bisturí. El objetivo del mismo.

## ¿Cuándo y para qué se usa?

Cada vez que quieras tocar un sistema en producción: instalar o actualizar un módulo, cambiar una configuración importante, importar datos en masa, ensayar una migración de versión o diagnosticar un error que solo se reproduce con los datos reales.

## Lo mínimo que necesitas saber

**1. No pruebes en producción: prueba en una copia**

La primera opción **nunca** es producción. Duplica la base de datos, neutralízala y prueba ahí (ver [Duplicar y neutralizar la base de datos](Duplicar-y-Neutralizar-Base-de-Datos.md)). Solo si algo no se puede reproducir fuera, se considera tocar producción, y con las cautelas siguientes.

**2. Backup antes de tocar nada**

Antes de cualquier cambio en producción, haz una copia de seguridad completa y **comprueba que puedes restaurarla** (ver [Backups y restauración](Backups-y-Restauracion.md)). Un cambio sin backup previo es un cambio del que quizá no puedas volver.

**3. Neutraliza los efectos hacia el exterior**

El mayor peligro de probar con datos reales es que "escapen" al mundo: correos, pagos, integraciones. En una copia, esto lo hace el comando `neutralize`. Si por fuerza pruebas en producción, ten especial cuidado con las acciones que envían correo o disparan cobros.

**4. Elige el momento y avisa**

Los cambios delicados se hacen en **ventanas de bajo uso** (fuera del horario laboral) y avisando a quien corresponda. Así, si algo se cae, no lo hace en plena jornada con todo el mundo trabajando.

**5. Cambios pequeños y reversibles, uno a uno**

Aplica un cambio cada vez y comprueba que todo sigue bien antes del siguiente. Si algo falla, sabrás exactamente qué lo causó. Los cambios grandes en bloque son mucho más difíciles de diagnosticar y deshacer.

**6. Solo lectura cuando solo necesites mirar**

Para investigar o consultar, usa un usuario con permisos de solo lectura o el modo desarrollador únicamente para *ver* (ver [Modo desarrollador](Modo-Desarrollador.md)). No hace falta poder escribir para diagnosticar.

**7. Nunca toques PostgreSQL a mano en producción**

Cambiar datos con `UPDATE`/`DELETE` directos sobre la base de datos salta toda la lógica de Odoo y puede dejar el sistema en un estado incoherente. Los cambios se hacen siempre a través de Odoo.

**8. Ten un plan de vuelta atrás**

Antes de empezar, ten claro cómo deshacer: qué backup restaurar, cómo desinstalar el módulo, a quién avisar. Si no sabes cómo revertir un cambio, aún no estás listo para hacerlo.

## Lo que NO hace

- **No garantiza cero riesgo** — reduce el riesgo y lo hace reversible, pero producción siempre merece respeto.
- **No sustituye a entender lo que haces** — la prudencia no arregla un cambio que no comprendes.
- **No es opcional "porque tengo prisa"** — saltarse el backup o la copia es justo cuando ocurren los desastres.

---

*En resumen: prueba siempre en una copia neutralizada, haz backup antes de tocar producción, aplica cambios pequeños y reversibles en horas tranquilas, y no empieces nada sin saber cómo deshacerlo.*
