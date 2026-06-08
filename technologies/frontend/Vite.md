# Vite

**¿Qué es?** Vite es una herramienta de build y servidor de desarrollo para proyectos frontend modernos. Empaqueta tu código React para producción y levanta un servidor local ultrarrápido mientras desarrollas.

---

## ¿Por qué existe?

Antes de Vite, herramientas como Webpack tardaban decenas de segundos en arrancar porque procesaban y empaquetaban todos los archivos del proyecto antes de servir nada. Conforme el proyecto crecía, el servidor tardaba más y más.

Analogía .NET: imagina que cada vez que haces un cambio en un endpoint, tu API tuviera que recompilar *toda* la solución antes de reflejar ese cambio, incluso cuando solo tocaste un controller. Eso era Webpack. Vite, en cambio, sirve los archivos directamente al navegador usando módulos ES nativos y solo recompila el archivo que cambió — como un hot reload granular.

---

## ¿Cómo encaja en este proyecto?

En **EcoWaveProjectManagement**, Vite cumple dos roles:

1. **Dev server**: cuando ejecutas `npm run dev`, Vite levanta el frontend en `localhost:5173` y conecta con la API .NET en otro puerto.
2. **Build de producción**: cuando ejecutas `npm run build`, Vite genera los archivos estáticos optimizados en `/dist`, que luego el backend .NET puede servir o desplegar por separado.

La configuración vive en `vite.config.ts`, en la raíz del proyecto frontend.

---

## Lo mínimo que necesitas saber

**1. El archivo de configuración**

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': 'http://localhost:5000' // redirige llamadas al backend .NET
    }
  }
})
```

**2. El proxy hacia el backend**

La clave `proxy` evita problemas de CORS en desarrollo: cualquier llamada a `/api/...` desde el frontend se redirige automáticamente al servidor .NET, sin tocar la URL en el código React.

**3. Variables de entorno**

Vite usa archivos `.env`. Solo las variables con prefijo `VITE_` son accesibles desde el código cliente:

```bash
# .env.local
VITE_API_BASE_URL=http://localhost:5000
```

```ts
const url = import.meta.env.VITE_API_BASE_URL
```

**4. Comandos esenciales**

```bash
npm run dev      # arranca el servidor de desarrollo
npm run build    # genera los archivos de producción en /dist
npm run preview  # sirve localmente el build de producción
```

---

## Lo que NO hace

- **No es un framework**: Vite no decide cómo estructuras componentes ni maneja rutas. Eso es React y React Router.
- **No gestiona el estado**: nada de stores ni contextos; eso lo hacen otras librerías.
- **No reemplaza al backend**: Vite solo sirve el frontend. Tu API .NET sigue siendo independiente.
- **No se usa en producción directamente**: en producción sirves los archivos de `/dist`, no el servidor Vite.

---

*En resumen: Vite es la herramienta que convierte tu código React en algo que el navegador puede ejecutar, tanto en desarrollo (rápido y con hot reload) como en producción (optimizado y empaquetado).*
