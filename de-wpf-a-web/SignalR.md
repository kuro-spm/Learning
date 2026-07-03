# SignalR

## ¿Qué es?

SignalR es una librería de ASP.NET Core para comunicación **en tiempo real**: permite que
el servidor **empuje** datos al cliente en cuanto ocurren, sin que el cliente tenga que
preguntar una y otra vez. Mantiene una conexión viva entre navegador y servidor para
enviar mensajes en ambos sentidos al instante.

## ¿Por qué existe?

[HTTP](HTTP.md) funciona por pregunta-respuesta: el cliente pide y el servidor responde,
pero el servidor **no puede iniciar** la conversación. Si quieres mostrar algo "que acaba
de pasar" (un mensaje nuevo, un pedido entrante, una notificación), con HTTP normal solo
te queda preguntar repetidamente "¿hay novedades?", lo cual es ineficiente y va con
retraso. SignalR resuelve esto manteniendo un canal abierto por el que el servidor avisa
en el momento.

> Si vienes de WPF, esto te sonará a los eventos y al patrón observador: en escritorio,
> cuando un dato cambiaba, disparabas un evento (`PropertyChanged`) y la UI reaccionaba al
> instante. SignalR trae esa idea a la web: el servidor "lanza un evento" y los navegadores
> conectados reaccionan, aunque estén en otra máquina.

## ¿Cuándo y para qué se usa?

Siempre que la pantalla deba actualizarse sola, sin que la usuaria recargue: un chat, las
notificaciones de una app, un marcador en directo, un panel que muestra pedidos entrantes
en una pantalla de cocina, o varios usuarios editando algo a la vez. Por ejemplo, en una
tienda online, avisar al panel de administración "ha entrado un pedido nuevo" en cuanto
ocurre, en lugar de que el equipo refresque la página cada minuto.

## Lo mínimo que necesitas saber

**1. El servidor define un *Hub*: el punto de encuentro en tiempo real**

Un *Hub* es una clase con métodos que los clientes pueden llamar y desde la que el
servidor puede llamar a los clientes:

```csharp
public class NotificacionesHub : Hub
{
    // Un cliente puede invocar esto...
    public async Task EnviarMensaje(string texto)
    {
        // ...y el servidor reenvía a TODOS los conectados
        await Clients.All.SendAsync("RecibirMensaje", texto);
    }
}
```

**2. Se registra y se publica como el resto de ASP.NET Core**

En `Program.cs`, igual que mapeas controladores, mapeas el hub a una ruta (ver [Routing y
Middleware](Routing-y-Middleware.md)):

```csharp
builder.Services.AddSignalR();
// ...
app.MapHub<NotificacionesHub>("/hub/notificaciones");
```

**3. El servidor puede empujar a todos, a un grupo o a un cliente concreto**

Esta es la potencia frente a HTTP normal: el servidor decide a quién avisa, cuando quiera:

```csharp
await Clients.All.SendAsync("RecibirMensaje", texto);        // a todos
await Clients.Group("Cocina").SendAsync("NuevoPedido", p);   // a un grupo
await Clients.Client(conexionId).SendAsync("Aviso", dato);   // a uno solo
```

**4. El cliente se suscribe a los mensajes y reacciona**

Desde el navegador (con JavaScript) o desde [Blazor](Blazor.md) en C#, el cliente abre la
conexión y queda a la escucha. Cuando llega un mensaje, actualiza la pantalla:

```javascript
const conexion = new signalR.HubConnectionBuilder()
    .withUrl("/hub/notificaciones")
    .build();

conexion.on("RecibirMensaje", (texto) => {
    // se ejecuta en cuanto el servidor empuja un mensaje
    mostrarEnPantalla(texto);
});

await conexion.start();
```

**5. Elige el mejor transporte por debajo, de forma transparente**

SignalR usa *WebSockets* cuando es posible (un canal bidireccional permanente) y, si el
entorno no lo permite, recurre automáticamente a otras técnicas. Tú trabajas con hubs y
mensajes; los detalles de transporte los gestiona la librería.

## Lo que NO hace

- **No reemplaza a HTTP ni a tu Web API** — convive con ellos; SignalR es para avisos en tiempo real, no para el CRUD normal.
- **No guarda los mensajes** — si necesitas un historial, lo persistes tú (por ejemplo con [EF Core](Entity-Framework-Core.md)).
- **No garantiza una conexión eterna** — las conexiones pueden caerse; hay que contemplar reconexiones y que un cliente pueda perderse mensajes.

## Buenas prácticas avanzadas

- **El hub es efímero: no guardes estado en sus campos** — SignalR crea una instancia nueva del hub *para cada invocación* y la destruye al terminar, como si cada mensaje estrenara objeto. Un campo `private List<string> _conectados` se vacía entre llamada y llamada, y el fallo es traicionero porque a veces "parece" funcionar. El estado compartido va fuera del hub: en grupos de SignalR, en un servicio singleton o en un almacén externo.
- **La reconexión no restaura los grupos** — `withAutomaticReconnect()` en el cliente es imprescindible (las conexiones se caen: wifi, proxies, suspensión del portátil), pero al reconectar el cliente recibe un `connectionId` **nuevo** y pierde sus membresías de grupo. Suscríbete al evento `onreconnected` y vuelve a pedirle al servidor que te meta en tus grupos; si no, tras el primer corte de red la usuaria deja de recibir avisos en silencio.
- **Con más de un servidor necesitas un *backplane*** — cada servidor solo conoce **sus** conexiones: si el pedido entra por el servidor A y la pantalla de cocina está conectada al B, `Clients.All` del A no le llega. En cuanto haya balanceador y varias instancias, hace falta un *backplane* (Redis) o el servicio gestionado Azure SignalR para que los mensajes crucen entre servidores. Funciona perfecto en local con una instancia y "pierde mensajes" en producción: este es el motivo.
- **Empuja avisos, no datos: "notifica y consulta"** — el patrón robusto es que SignalR envíe mensajes pequeños ("hay un pedido nuevo", con su id) y el cliente pida el detalle por la Web API de siempre. Así los mensajes perdidos importan menos (la próxima consulta trae la verdad), evitas mandar objetos gordos por el canal en tiempo real y toda la lógica de datos sigue en un único sitio: tu API.

---

*En resumen: SignalR lleva el patrón de eventos del escritorio a la web —el servidor empuja datos a los navegadores conectados en tiempo real— resolviendo lo que HTTP por sí solo no puede: avisar sin que se lo pidan.*
