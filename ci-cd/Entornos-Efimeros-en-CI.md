# Entornos efímeros en CI

**¿Qué es?** Un *entorno efímero* es una copia completa de la aplicación (servicios, base de datos, todo) que el pipeline **levanta desde cero, usa para probar y destruye al terminar** — normalmente con contenedores Docker dentro del propio runner de CI. No existe antes del test ni después.

---

## ¿Por qué existe?

La forma clásica de probar "la aplicación entera" es mantener un servidor de TEST siempre encendido donde se despliega antes que a producción. Ese servidor tiene costes ocultos: hay que mantenerlo, pagar su infraestructura, y con el tiempo se convierte en un "blanco móvil" — datos viejos, versiones a medias, configuración que ya no coincide con producción. Los tests fallan por culpa del entorno, no del código.

El entorno efímero elimina el problema de raíz: si cada ejecución **construye su propio mundo limpio**, no hay nada que se desactualice ni estado de una ejecución que contamine la siguiente.

> Es la diferencia entre ensayar en un teatro compartido (donde el decorado queda como lo dejó el último grupo) y montar tu decorado nuevo cada vez, idéntico al del estreno.

---

## ¿Cuándo y para qué se usa?

- **Tests de integración con base de datos real:** en lugar de un servidor de BD compartido, el test arranca un PostgreSQL en contenedor, aplica el esquema y lo destruye al acabar (librerías como **TestContainers** automatizan esto desde el propio código de test).
- **Tests E2E sin entorno desplegado:** el pipeline hace `docker compose up` con la app completa (API + web + BD) dentro del runner y lanza la suite (Playwright, Cypress) contra `localhost`.
- **Preview environments:** algunos equipos levantan un entorno efímero por cada PR para revisarlo visualmente, y lo destruyen al mergear.

---

## Lo mínimo que necesitas saber

**1. Levantar el stack en el runner y probar contra él**

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose up -d --build        # app completa en el runner
      - run: npx wait-on http://localhost:8080/health
      - run: npx playwright test                 # E2E contra localhost
      - run: docker compose down -v              # el entorno desaparece
```

**2. Base de datos efímera desde el código de test (TestContainers)**

```csharp
// El test arranca su propio PostgreSQL y lo destruye al terminar
var db = new PostgreSqlBuilder("postgres:16-alpine").Build();
await db.StartAsync();
// ... aplicar esquema, ejecutar el test contra db.GetConnectionString() ...
await db.DisposeAsync();
```

**3. La clave: partir siempre de estado cero**

Cada ejecución siembra sus propios datos de prueba. Si un test necesita un usuario, lo crea; nunca asume que "ya existe" de una ejecución anterior.

---

## Lo que NO hace

- **No prueba la infraestructura real** — nginx, DNS, certificados o la configuración del servidor de producción quedan fuera; eso solo lo cubre un *smoke test* tras el deploy real.
- **No es gratis en tiempo** — construir y arrancar el mundo en cada ejecución añade minutos; se compensa con cachés de imágenes y ejecutándolo solo en los puntos que tocan (no en cada commit).
- **No sirve para probar con datos reales de producción** — los datos son sembrados; probar migraciones sobre volúmenes reales requiere otra estrategia.

---

## Buenas prácticas avanzadas

- **Clava las versiones a las de producción** — el valor del entorno efímero es que se parezca al real: si producción corre `postgres:16.3`, el contenedor de test debe ser `postgres:16.3`, no `postgres:latest`. Con `latest`, el día que la imagen salta de versión mayor tus tests validan una base de datos que no es la tuya — y encima el fallo aparece "sin que nadie haya cambiado nada".
- **Vuelca los logs del stack cuando falle** — el entorno se destruye al terminar, así que un E2E rojo sin más información es indepurable: el error real suele estar en los logs de la API o de la BD, que ya no existen. Añade un paso condicionado al fallo que los publique antes del `down`.

  ```yaml
  - run: docker compose logs --timestamps
    if: failure()          # solo si algo falló; se guarda antes de destruir
  ```
- **El runner nace vacío: trae la caché de fuera** — como cada ejecución parte de una máquina limpia, el `docker compose up --build` reconstruye todas las capas desde cero y los minutos se disparan. Usa `--cache-from` apuntando a la última imagen publicada en el registro (o la caché de imágenes de tu plataforma de CI) para que solo se reconstruyan las capas que de verdad cambiaron.

---

*En resumen: un entorno efímero cambia "mantener un servidor de pruebas siempre a medio morir" por "construir un mundo limpio, probarlo y tirarlo" — la reproducibilidad se consigue no dejando nada vivo entre ejecuciones.*
