# pnpm

**¿Qué es?** Un gestor de paquetes para proyectos JavaScript/TypeScript que reemplaza a npm o yarn, siendo significativamente más rápido y eficiente en el uso del disco.

---

## ¿Por qué existe?

npm (el gestor predeterminado de Node.js) tiene un problema clásico: cada proyecto descarga y copia sus propias dependencias en una carpeta `node_modules` local. Si tienes tres proyectos que usan React, tienes tres copias de React en tu máquina.

Es el equivalente a que cada microservicio de tu solución .NET descargara sus propios binarios de `Newtonsoft.Json` en una carpeta local en lugar de usar el caché global de NuGet. pnpm soluciona esto con un almacén global de paquetes y enlaces simbólicos: cada versión de cada paquete se descarga una sola vez y se comparte entre proyectos.

El resultado es instalaciones mucho más rápidas y un consumo de disco notablemente menor.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, pnpm gestiona todas las dependencias del frontend en React. Al clonar el repositorio, ejecutas pnpm para instalar los paquetes necesarios (React, librerías de UI, herramientas de build, etc.) antes de arrancar la aplicación en local o en CI/CD.

El archivo `pnpm-lock.yaml` (equivalente al `packages.lock.json` de NuGet) garantiza que todos los miembros del equipo y los entornos de despliegue usen exactamente las mismas versiones de cada dependencia.

---

## Lo mínimo que necesitas saber

**1. Instalar dependencias del proyecto**
```bash
pnpm install
```
Ejecuta esto una vez al clonar el repo o cuando alguien añada un nuevo paquete.

**2. Añadir una nueva dependencia**
```bash
pnpm add axios
pnpm add -D @types/node   # dependencia de desarrollo (equivale a devDependencies)
```

**3. Ejecutar scripts definidos en `package.json`**
```bash
pnpm dev      # arranca el servidor de desarrollo
pnpm build    # genera el build de producción
pnpm lint     # ejecuta el linter
```

**4. El archivo `package.json` es el punto de entrada**
```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "dependencies": {
    "react": "^18.2.0"
  }
}
```
Es el análogo al `.csproj` de tu proyecto .NET: declara dependencias y comandos.

---

## Lo que NO hace

- No compila ni transpila TypeScript/JSX por sí solo — eso lo hace Vite o el compilador de TypeScript.
- No gestiona versiones de Node.js — para eso existe `nvm` o `fnm`.
- No tiene relación con los paquetes NuGet del backend; son ecosistemas completamente separados.
- No reemplaza a Docker ni a ninguna herramienta de despliegue.

---

*En resumen: pnpm es la herramienta que usas para instalar y gestionar las librerías del frontend, de forma más rápida y ordenada que npm; una vez instaladas las dependencias, raramente necesitarás pensar en él.*
