# TanStack Query (React Query)

**¿Qué es?** Una librería para React que gestiona el ciclo de vida completo de los datos que vienen del servidor: los pide, los guarda en caché, los sincroniza y los actualiza automáticamente.

---

## ¿Por qué existe?

En el backend, cuando haces una llamada a base de datos, no repites la query en cada método que la necesita: usas caché, repositorios o servicios compartidos. En el frontend sin TanStack Query, cada componente que necesita datos hace su propio `fetch`, gestiona su propio estado de carga y error, y no sabe nada de lo que han pedido los demás componentes.

El resultado es peticiones duplicadas, estados inconsistentes y código lleno de `useEffect` + `useState` para cosas que deberían ser simples. TanStack Query resuelve exactamente eso: centraliza la gestión del dato remoto, igual que un servicio con caché en el backend.

---

## ¿Cuándo y para qué se usa?

Aparece en cualquier frontend React que consuma una API REST (construida en C#/.NET o cualquier otro backend). TanStack Query actúa como la capa que:

- Llama a los endpoints del backend (productos, pedidos, usuarios...).
- Guarda los resultados en caché para no repetir llamadas innecesarias.
- Invalida y refresca la caché automáticamente cuando se crea o modifica un recurso.

La configuración global vive en el punto de entrada de la app, y cada feature usa sus propios hooks de consulta.

---

## Lo mínimo que necesitas saber

**1. `useQuery` — leer datos del servidor**

```tsx
const { data, isLoading, isError } = useQuery({
  queryKey: ['products'],
  queryFn: () => fetch('/api/products').then(res => res.json()),
});
```

`queryKey` es el identificador de caché. Si dos componentes usan la misma key, comparten el dato — solo se hace una petición.

**2. `useMutation` — escribir datos (POST, PUT, DELETE)**

```tsx
const mutation = useMutation({
  mutationFn: (newProduct) => fetch('/api/products', { method: 'POST', body: JSON.stringify(newProduct) }),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['products'] }),
});
```

`invalidateQueries` es el equivalente a "marcar la caché como sucia" para que se recargue.

**3. `QueryClient` — el contenedor central de caché**

```tsx
const queryClient = new QueryClient();

// En el punto de entrada:
<QueryClientProvider client={queryClient}>
  <App />
</QueryClientProvider>
```

Toda la app comparte este cliente. Es el equivalente a un servicio singleton en .NET.

---

## Lo que NO hace

- No es un cliente HTTP — necesitas `fetch`, `axios` u otro para hacer la petición real.
- No gestiona estado de UI local (formularios, modales, toggles) — para eso existe el estado de React o Zustand.
- No reemplaza al backend: no persiste datos, solo los sincroniza desde la API.

---

*En resumen: TanStack Query es la capa de caché y sincronización entre tu frontend React y la API .NET — evita peticiones duplicadas, mantiene los datos consistentes y elimina el boilerplate de `useEffect` para cargar datos.*
