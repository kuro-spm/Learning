# Jenkins

**¿Qué es?** Jenkins es un servidor de automatización de CI/CD de código abierto que **tú instalas y administras** en tu propia máquina o servidor. Es una de las herramientas más veteranas y extensibles del mundo CI/CD.

---

## ¿Por qué existe?

Herramientas como GitHub Actions o GitLab CI viven dentro de una plataforma concreta y se ejecutan en su nube. Jenkins nace antes que ellas y toma otro enfoque: es un programa independiente que instalas donde quieras y conectas al repositorio que sea. A cambio de más trabajo de configuración, te da **control total** y un catálogo enorme de plugins para integrarse con casi cualquier herramienta.

> Si ya conoces GitHub Actions, piensa en Jenkins como "el CI/CD que no depende de ninguna plataforma: tú pones el servidor y decides todo, pero también lo mantienes tú".

---

## ¿Cuándo y para qué se usa?

Suele aparecer en organizaciones que necesitan ejecutar su CI/CD en infraestructura propia (por seguridad, normativa o porque ya la tienen):

- Una **tienda online** que debe construirse en servidores internos, sin salir a la nube pública.
- Proyectos que usan herramientas muy específicas para las que existe un plugin de Jenkins.
- Equipos que ya tienen Jenkins funcionando desde hace años y mantienen sus pipelines ahí.

---

## Lo mínimo que necesitas saber

**1. Es un servidor que se instala**

A diferencia de las soluciones en la nube, alguien tiene que instalar, actualizar y mantener Jenkins. Se accede a él por una interfaz web.

**2. El Jenkinsfile**

El pipeline moderno se define en un archivo llamado `Jenkinsfile` en la raíz del repositorio, usando un lenguaje propio (basado en Groovy).

```groovy
pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                sh 'npm install'
                sh 'npm test'
            }
        }
        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

**3. Plugins**

Casi toda la potencia de Jenkins viene de sus miles de *plugins*: para conectar con Git, Docker, Slack, sistemas de notificación, etc. Se instalan desde la propia interfaz.

**4. Agentes (nodos)**

Jenkins puede repartir el trabajo entre varias máquinas (*agentes*), por ejemplo para compilar en Windows y en Linux a la vez.

---

## Lo que NO hace

- **No es "plug and play"** — requiere instalación y mantenimiento, a diferencia de las soluciones integradas en GitHub o GitLab.
- **No está atado a una plataforma** — funciona con cualquier repositorio, pero esa libertad implica configurarlo tú.
- **No se actualiza solo** — las versiones y los plugins los gestiona quien administra el servidor.

---

## Buenas prácticas avanzadas

- **Nada configurado a golpe de clic** — un Jenkins montado desde la interfaz web es irreproducible: si el servidor muere, nadie recuerda qué casillas estaban marcadas. Los equipos expertos definen los pipelines solo en el `Jenkinsfile` y la configuración del propio servidor con *Configuration as Code* (plugin JCasC), de modo que todo Jenkins se puede reconstruir desde archivos versionados.
- **El controller no ejecuta builds** — la máquina principal de Jenkins debe tener sus *executors* a 0 y delegar todo el trabajo en agentes. Si los builds corren en el controller, cualquier pipeline puede leer credenciales y archivos del propio servidor, y un build pesado puede tumbar la interfaz para todo el equipo.
- **Cada plugin es una deuda, no un regalo** — el catálogo enorme es la mayor fortaleza de Jenkins y su mayor fuente de roturas: los plugins se actualizan a ritmos distintos y una actualización puede romper otro plugin o el propio servidor. Instala los mínimos imprescindibles y prueba las actualizaciones en una instancia de ensayo antes de tocar la real.

---

*En resumen: Jenkins es el veterano del CI/CD — un servidor propio, infinitamente extensible con plugins, que da control total a cambio de tener que instalarlo y mantenerlo.*
