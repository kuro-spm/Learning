# GitHub Actions

## ¿Qué es?

GitHub Actions es el sistema de **automatización e integración continua (CI/CD)** de GitHub. Permite ejecutar tareas automáticamente cuando ocurre algo en tu repositorio: por ejemplo, lanzar los tests cada vez que alguien sube código o abre un pull request.

## ¿Por qué existe?

Hay tareas que deberían pasar *siempre*, pero que es fácil olvidar: ejecutar los tests, revisar el estilo del código, construir la aplicación, desplegarla. Hacerlas a mano es lento y poco fiable. GitHub Actions las automatiza: defines una vez "cuando pase X, haz Y" y se ejecuta solo, en los servidores de GitHub, sin que nadie tenga que acordarse.

Esto es la base de **CI/CD**:
- **CI (Integración Continua)**: cada cambio se prueba automáticamente al integrarse, para detectar errores cuanto antes.
- **CD (Entrega/Despliegue Continuo)**: si todo pasa, el cambio se publica automáticamente.

> Imagina un asistente que, cada vez que alguien entrega trabajo, comprueba sin falta que no rompe nada antes de aceptarlo. GitHub Actions es ese asistente, y no se cansa ni se olvida.

## ¿Cuándo y para qué se usa?

- **Ejecutar tests** automáticamente en cada push o pull request.
- **Revisar el estilo** del código (linters) antes de fusionar.
- **Construir y desplegar** una aplicación cuando se publica una versión.

## Lo mínimo que necesitas saber

**1. Los flujos de trabajo viven en una carpeta concreta**

Se definen en archivos `.yml` dentro de `.github/workflows/` en tu repositorio. Cada archivo es un *workflow*.

**2. Anatomía de un workflow**

```yaml
# .github/workflows/tests.yml
name: Tests

on: [push, pull_request]   # CUÁNDO se ejecuta

jobs:
  test:                    # QUÉ hace
    runs-on: ubuntu-latest # en qué máquina
    steps:
      - uses: actions/checkout@v4   # descarga tu código
      - run: npm install            # instala dependencias
      - run: npm test               # ejecuta los tests
```

**3. Los conceptos clave**

- **Evento (`on`)**: qué dispara el workflow (un push, un pull request, una hora del día...).
- **Job**: un conjunto de pasos que se ejecuta en una máquina limpia.
- **Step**: cada acción concreta; puede ser un comando (`run`) o una acción reutilizable (`uses`).

**4. Dónde ver los resultados**

En la pestaña **Actions** del repositorio en GitHub. Si algo falla, un ✅ se convierte en ❌ y normalmente bloquea la fusión del pull request.

## Lo que NO hace

- **No es parte de Git** — es un servicio de GitHub. Otras plataformas tienen su equivalente (GitLab CI, etc.).
- **No adivina qué quieres automatizar** — tú defines los pasos en el `.yml`.
- **No es gratis sin límite** — el tiempo de ejecución tiene cuotas, generosas para proyectos públicos pero limitadas en privados.

---

*En resumen: GitHub Actions es el asistente incansable que ejecuta tus tests y despliegues automáticamente con cada cambio — el puente entre "subí el código" y "el código funciona y está publicado".*
