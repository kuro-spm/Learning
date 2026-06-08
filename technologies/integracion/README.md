# Cómo se comunican el frontend y el backend

## Visión general

```
Navegador
    │
    ▼
Frontend React/Vite  :5173
    │  (proxy /api → localhost:5000)
    ▼
Backend ASP.NET Core  :5000
    │
    ▼
Base de datos PostgreSQL  :5432
```

El navegador solo habla con el frontend. El frontend es el único que habla con el backend. El backend es el único que habla con la base de datos. Esta separación en capas facilita el mantenimiento y la seguridad.

---

## En desarrollo local

### Cómo funciona el proxy de Vite

Cuando el componente React hace `fetch('/api/projectes')`, Vite intercepta esa petición y la reenvía automáticamente a `http://localhost:5000/api/projectes`. Esto se llama **proxy inverso en desarrollo**.

El proxy está configurado en `vite.config.ts`:

```ts
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
      },
    },
  },
})
```

**Por qué existe:** Sin el proxy, el navegador bloquearía las peticiones del frontend (`localhost:5173`) al backend (`localhost:5000`) por la política de seguridad CORS. El proxy hace que, desde el punto de vista del navegador, todo venga del mismo origen (`localhost:5173`).

**Por qué el frontend nunca llama directamente a `localhost:5000`:** Porque las URLs en el código siempre usan rutas relativas como `/api/projectes`, no URLs absolutas. Así el mismo código funciona tanto en desarrollo (con el proxy de Vite) como en producción (con Docker).

### Arrancar el proyecto completo

```bash
# Terminal 1 — Backend
cd backend
dotnet run

# Terminal 2 — Frontend
cd frontend
pnpm dev
```

El backend queda disponible en `http://localhost:5000` y el endpoint de salud en `http://localhost:5000/healthz`. El frontend en `http://localhost:5173`.

---

## En producción (Docker)

### Cómo Docker Compose conecta los tres servicios

Docker Compose crea una red interna donde los servicios se comunican por nombre de servicio (`backend`, `frontend`, `db`). El frontend llama al backend usando `http://backend:5000` dentro de esa red, de forma transparente.

```yaml
# docker-compose.yml (fragmento ilustrativo)
services:
  backend:
    ports: ["5000:5000"]
  frontend:
    ports: ["5173:5173"]
  db:
    image: postgres
```

### CORS: qué es y por qué importa

CORS (Cross-Origin Resource Sharing) es un mecanismo de seguridad del navegador. Cuando el frontend y el backend están en dominios o puertos distintos, el navegador exige que el backend diga explícitamente "acepto peticiones de ese origen".

En producción, el backend tiene CORS configurado en `Program.cs`:

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("FrontendPolicy", policy =>
        policy.WithOrigins("http://localhost:5173")   // origen permitido
              .AllowAnyMethod()
              .AllowAnyHeader());
});

app.UseCors("FrontendPolicy");
```

---

## El flujo de una petición, paso a paso

**Escenario:** el usuario abre la lista de proyectos.

1. **React + TanStack Query** — el componente monta y TanStack Query ejecuta la consulta:

```ts
const { data } = useQuery({
  queryKey: ['projectes'],
  queryFn: () => fetch('/api/projectes').then(r => r.json()),
})
```

2. **Proxy de Vite** — intercepta `/api/projectes` y la convierte en `http://localhost:5000/api/projectes`.

3. **ASP.NET Core** — el `ProjectesController` recibe la petición, llama a la capa de aplicación, que usa Dapper para ejecutar la consulta SQL sobre PostgreSQL.

```csharp
[HttpGet]
public async Task<IActionResult> GetAll()
{
    var projectes = await _projectesService.GetAllAsync();
    return Ok(projectes);
}
```

4. **Respuesta JSON** — el backend serializa la lista y devuelve `200 OK` con JSON. TanStack Query cachea el resultado y React re-renderiza el componente con los datos.

---

## Cómo añadir un nuevo endpoint (guía rápida)

1. **Backend — crear el endpoint:**

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetById(int id)
{
    var projecte = await _projectesService.GetByIdAsync(id);
    return projecte is null ? NotFound() : Ok(projecte);
}
```

2. **Verificar con curl o Swagger** (`http://localhost:5000/swagger`):

```bash
curl http://localhost:5000/api/projectes/1
```

3. **Frontend — consumirlo con TanStack Query:**

```ts
const { data } = useQuery({
  queryKey: ['projecte', id],
  queryFn: () => fetch(`/api/projectes/${id}`).then(r => r.json()),
})
```

---

## Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `CORS error` en producción | El origen del frontend no está en la lista blanca del backend | Añadir el dominio exacto en `WithOrigins(...)` en `Program.cs` |
| `404 Not Found` en una ruta nueva | La URL no incluye el prefijo `/api` | Verificar que el Controller tiene `[Route("api/[controller]")]` y que el fetch usa `/api/...` |
| Petición va a `:5173` en vez del backend | Se usó una URL absoluta en el fetch en lugar de una ruta relativa | Cambiar `fetch('http://localhost:5000/api/...')` por `fetch('/api/...')` |

> **Autenticación:** no está implementada en la versión actual del proyecto. Se añadirá en una iteración futura (JWT o similar).
