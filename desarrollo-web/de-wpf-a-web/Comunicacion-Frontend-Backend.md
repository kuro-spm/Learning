# Comunicación entre frontend y backend

## ¿Qué es?

Es el conjunto de piezas —proxy de desarrollo, CORS, orquestación con contenedores— que hacen que un frontend SPA y una Web API backend, ejecutándose en procesos y puertos distintos, puedan hablar entre sí tanto en desarrollo como en producción.

## ¿Por qué existe?

En WPF la interfaz y la lógica vivían en el mismo proceso: llamar a un servicio era una llamada a un método en memoria. En web, el frontend (servido por su propio servidor de desarrollo) y el backend (otra aplicación, en otro puerto) son dos programas independientes que se comunican por HTTP a través de la red, incluso en tu propia máquina. Eso trae dos problemas nuevos: cómo evitar escribir URLs absolutas por todas partes, y cómo lidiar con la política de seguridad del navegador que bloquea peticiones entre orígenes distintos.

> Si vienes de WPF: es como si tu ventana y tu servicio de negocio vivieran en dos procesos distintos y tuvieran que hablar por red en lugar de con una llamada directa en memoria.

## ¿Cuándo y para qué se usa?

Aparece en cualquier aplicación con un frontend SPA (React, Angular, Vue...) y un backend independiente: una tienda online, un panel de administración, una app de gestión de tareas. Se necesita tanto en desarrollo local, para que el navegador no bloquee las peticiones, como en producción, para que los contenedores del frontend y el backend se encuentren en la red.

## Lo mínimo que necesitas saber

**1. El proxy de desarrollo evita URLs absolutas**

Herramientas como Vite pueden interceptar peticiones a `/api/*` en desarrollo y reenviarlas al backend real, para que el código del frontend nunca necesite conocer su host o puerto:

```ts
// vite.config.ts
export default defineConfig({
  server: {
    proxy: { '/api': { target: 'http://localhost:5000', changeOrigin: true } },
  },
})
```

Así el `fetch` en el componente siempre usa una ruta relativa:

```ts
fetch('/api/productos')
```

**2. CORS autoriza (o bloquea) las peticiones entre orígenes**

Cuando frontend y backend viven en puertos u hosts distintos, el navegador exige que el backend declare explícitamente qué orígenes puede aceptar:

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("FrontendPolicy", policy =>
        policy.WithOrigins("http://localhost:5173")
              .AllowAnyMethod()
              .AllowAnyHeader());
});

app.UseCors("FrontendPolicy");
```

**3. En producción, los contenedores se hablan por nombre de servicio**

Con Docker Compose, cada servicio tiene un nombre dentro de la red interna; el frontend llama al backend usando ese nombre en lugar de `localhost`:

```yaml
services:
  backend:
    ports: ["5000:5000"]
  frontend:
    ports: ["5173:5173"]
```

**4. El flujo completo de una petición**

1. El componente pide datos con una ruta relativa (`fetch('/api/productos')`).
2. En desarrollo, el proxy la reenvía al backend; en producción, ya apunta al dominio real.
3. El controlador de la Web API la recibe, ejecuta la lógica y devuelve JSON.
4. El frontend recibe la respuesta y actualiza la interfaz.

## Lo que NO hace

- **CORS no es una medida de seguridad del servidor** — protege al usuario en el navegador, pero no impide que otro programa fuera del navegador llame a tu API directamente.
- **El proxy de desarrollo no existe en producción** — ahí la comunicación depende de cómo despliegues (dominio real, gateway, red de contenedores), no de la configuración de Vite.

## Buenas prácticas avanzadas

- **Nunca escribas la URL del backend en el código del frontend** — usa siempre rutas relativas y deja que el proxy (en desarrollo) o la configuración de despliegue (en producción) resuelvan el destino real. Si un día cambia el dominio del backend, no tocas ni una línea de frontend.
- **Sé explícito con los orígenes permitidos en CORS** — declarar dominios exactos con `WithOrigins(...)` en vez de `AllowAnyOrigin()` es lo que hace que la protección sirva de algo; `AllowAnyOrigin()` combinado con credenciales ni siquiera está permitido por la propia especificación CORS.

## Recursos didácticos divertidos

Para depurar peticiones CORS bloqueadas, la pestaña "Network" de las herramientas de desarrollador del navegador es tu mejor amiga: un error CORS siempre aparece ahí con el motivo exacto.

---

*En resumen: frontend y backend son dos programas separados que hablan por HTTP; el proxy de desarrollo y CORS son las dos piezas que hacen que esa conversación funcione tanto en tu máquina como en producción.*
