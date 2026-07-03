# CI/CD

## ¿Qué es?

CI/CD es una forma de trabajar (y un conjunto de herramientas) que **automatiza los pasos entre escribir código y ponerlo en producción**. Las siglas significan *Continuous Integration* (Integración Continua) y *Continuous Delivery/Deployment* (Entrega/Despliegue Continuo).

## ¿Por qué existe?

Sin CI/CD, cada vez que alguien termina una funcionalidad tiene que, a mano: descargar la última versión del código, ejecutar los tests, comprobar que nada se rompe, generar la versión final (*build*) y subirla al servidor. Es lento, fácil de olvidar pasos y muy propenso a errores humanos ("en mi máquina funcionaba").

CI/CD automatiza todo eso: un robot se encarga de repetir siempre los mismos pasos, de la misma forma, cada vez que el código cambia.

> Si vienes del mundo de la cocina: CI/CD es como una cadena de montaje en un restaurante. Tú dejas los ingredientes (el código), y la cadena se encarga de cocinar, emplatar y servir siempre igual, sin que se olvide la sal.

Conviene separar las tres ideas:

- **CI (Integración Continua):** cada cambio se fusiona con el código común a menudo, y automáticamente se compila y se pasan los tests. Así los errores aparecen pronto, no semanas después.
- **CD (Entrega Continua):** además de probar, se prepara una versión lista para desplegar en cualquier momento, pero el botón final lo pulsa una persona.
- **CD (Despliegue Continuo):** el paso más allá: si todo pasa, se despliega a producción **automáticamente**, sin intervención humana.

## ¿Cuándo y para qué se usa?

Aparece en casi cualquier proyecto de software con más de una persona o que se actualice con frecuencia:

- Una **tienda online** que despliega varias veces al día sin que la web se caiga.
- Un **blog** que publica el sitio cada vez que se actualiza un artículo en el repositorio.
- Una **app de tareas** que ejecuta toda su batería de tests antes de aceptar cualquier cambio nuevo.

Su objetivo es siempre el mismo: entregar software **más rápido, más seguro y con menos errores**.

## Lo mínimo que necesitas saber

**1. Todo arranca con un cambio en el repositorio**

El flujo se dispara solo cuando ocurre algo en Git: un `push`, abrir una *pull request*, crear una etiqueta de versión... La herramienta de CI/CD está "escuchando" esos eventos.

```yaml
# Ejecutar cuando se hace push a la rama main
on:
  push:
    branches: [main]
```

**2. CI: integrar y probar**

El robot descarga el código, instala dependencias, compila y pasa los tests. Si algo falla, avisa y bloquea el cambio.

```bash
npm install      # instalar dependencias
npm run build    # compilar
npm test         # ejecutar los tests
```

**3. CD: entregar o desplegar**

Si la fase de CI pasa, se genera el artefacto final (la app empaquetada) y se publica donde toque: un servidor, un servicio en la nube, una tienda de apps...

```bash
# Ejemplo simplificado de despliegue
scp app.zip usuario@servidor:/var/www/app
ssh usuario@servidor "systemctl restart app"
```

**4. Feedback rápido**

La clave de CI/CD es enterarse pronto. Si rompes algo, te avisa en minutos (con un check rojo en la pull request), no cuando ya está en producción.

## Lo que NO hace

- **No escribe los tests por ti** — solo los ejecuta. Si no hay tests, CI no detecta nada.
- **No garantiza que el código sea bueno** — garantiza que pasa los controles que tú hayas definido.
- **No sustituye las decisiones humanas** — alguien tiene que decidir qué se prueba, cuándo se despliega y qué hacer si falla.

## Buenas prácticas avanzadas

- **Integrar a menudo es la práctica; el pipeline es solo la herramienta** — un equipo con ramas que viven semanas no hace Integración Continua aunque tenga el mejor pipeline del mundo: los conflictos y las sorpresas se acumulan igual. La señal de que hay CI de verdad es que los cambios llegan a la rama común cada pocos días como mucho.
- **Build once, deploy many** — el artefacto que pasa los tests debe ser *exactamente* el que llega a producción. Si recompilas para cada entorno (uno para staging, otro para producción), estás desplegando algo que nunca se probó: cualquier diferencia de versión de compilador o dependencia se cuela sin que ningún test la vea. Compila una vez y promociona el mismo binario/imagen entre entornos.
- **Ordena el pipeline para fallar pronto** — coloca primero lo barato y lo que más veces falla (lint, compilación, tests unitarios) y al final lo caro (E2E, builds de Docker). Fallar en el segundo 30 en vez de en el minuto 20 multiplica los ciclos de trabajo que caben en un día.
- **Trata los tests intermitentes (flaky) como averías, no como ruido** — cuando un test falla "a veces" y la costumbre es relanzar hasta que sale verde, el equipo deja de creerse los rojos... y el día que el rojo es real, se ignora. Pon los tests flaky en cuarentena y arréglalos o bórralos: un test en el que no confías es peor que no tenerlo.
- **Vigila la duración total como una métrica de producto** — si el pipeline tarda más de ~10 minutos, la gente agrupa cambios grandes para "no esperar dos veces", y eso destruye justo lo que CI/CD quiere conseguir: cambios pequeños y frecuentes. Cuando el pipeline engorda, invertir en acelerarlo (paralelizar, cachear) es trabajo de primera clase, no una tarea de relleno.

---

*En resumen: CI/CD es una cadena de montaje automática para tu código — integra, prueba y despliega siempre de la misma forma, para que entregar software deje de dar miedo.*
