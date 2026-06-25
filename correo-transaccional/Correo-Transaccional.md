# Correo transaccional en .NET

## ¿Qué es?

El **correo transaccional** es el email que tu aplicación envía de forma automática como reacción a una acción concreta de una sola persona: confirmar un registro, recuperar una contraseña, avisar de que un pedido se ha enviado, mandar una factura. No es publicidad: es un mensaje uno-a-uno disparado por un evento del sistema.

## ¿Por qué existe?

Mandar un correo "de verdad" tiene dos partes que la gente confunde:

- **SMTP** es solo el *protocolo* (el idioma) con el que dos máquinas se pasan un correo. Que exista el protocolo no significa que tengas un servidor que lo hable ni que ese servidor tenga buena reputación.
- La **entregabilidad** es que el correo llegue de verdad a la bandeja de entrada, no a la carpeta de spam ni rechazado. Esto depende de la reputación de la IP desde la que envías y de tener bien configurado el dominio (SPF, DKIM, DMARC).

> Si vienes del mundo web, piensa en SMTP como en HTTP: tener el protocolo no te da un servidor. Igual que necesitas montar (o contratar) un servidor web para responder peticiones HTTP, necesitas un servidor de correo con reputación para que tus emails lleguen.

Por eso surgieron los **proveedores de correo transaccional**: servicios que ya tienen la infraestructura, la reputación de IP y la configuración de entregabilidad resueltas. Tú les pides "envía este correo" y ellos se encargan del resto.

## ¿Cuándo y para qué se usa?

Siempre que una app necesite avisar a un usuario concreto de algo que le ha pasado a *su* cuenta. Ejemplos genéricos:

- Una tienda online que manda el **email de confirmación de pedido** y el de "tu paquete va en camino".
- Una app de tareas que envía un **enlace para recuperar la contraseña**.
- Un formulario de registro que manda un **correo de verificación** ("haz clic para activar tu cuenta").
- Un blog que avisa por correo de **comentarios nuevos** en tus publicaciones.

Lo que **no** es correo transaccional: las newsletters y campañas de marketing (muchos destinatarios, mismo contenido). Para eso hay otras herramientas, aunque varios proveedores cubran ambos casos.

### ¿Y puedo asumir que ya existe un SMTP donde despliegue?

No conviene asumirlo. Tienes tres escenarios:

1. **Montar tu propio servidor SMTP** en la máquina donde despliegas. Técnicamente posible, pero tus correos acabarán casi siempre en spam: una IP nueva no tiene reputación. **No recomendado** salvo casos muy concretos.
2. **Usar el correo corporativo del cliente** (por ejemplo Microsoft 365 o Google Workspace). Funciona, pero tiene límites de envío y hoy suele exigir autenticación moderna (OAuth2) en lugar de usuario y contraseña.
3. **Usar un proveedor transaccional** (Amazon SES, SendGrid, Mailgun, Postmark, Brevo, Resend, Azure Communication Services...). Es lo recomendado: te dan credenciales SMTP **o** una API, y resuelven la entregabilidad por ti. La elección suele depender de dónde despliegas (SES si estás en AWS, Azure Communication Services si estás en Azure) y de la capa gratuita.

## Lo mínimo que necesitas saber

**1. No hardcodees la configuración: ponla en `appsettings`**

Host, puerto, usuario y contraseña (o API key) van en configuración, nunca en el código. Así cambiar de proveedor es cambiar texto, no recompilar.

```json
// appsettings.json
{
  "Email": {
    "Host": "smtp.proveedor.com",
    "Port": 587,
    "User": "apikey",
    "Password": "REEMPLAZAR_CON_SECRETO",
    "From": "no-reply@miapp.com",
    "FromName": "Mi App"
  }
}
```

> La contraseña o API key real **no** debe estar en `appsettings.json` dentro del repositorio. Usa *user secrets* en desarrollo y variables de entorno o un gestor de secretos (Azure Key Vault, AWS Secrets Manager) en producción.

**2. Enviar un correo por SMTP con MailKit**

El cliente SMTP recomendado en .NET es **MailKit** (la clase `SmtpClient` antigua de `System.Net.Mail` está desaconsejada por Microsoft). Se instala con `dotnet add package MailKit`.

```csharp
using MailKit.Net.Smtp;
using MimeKit;

var mensaje = new MimeMessage();
mensaje.From.Add(new MailboxAddress("Mi App", "no-reply@miapp.com"));
mensaje.To.Add(MailboxAddress.Parse("usuario@ejemplo.com"));
mensaje.Subject = "Recupera tu contraseña";
mensaje.Body = new TextPart("html")
{
    Text = "<p>Haz clic <a href=\"https://miapp.com/reset?token=abc\">aquí</a> para restablecerla.</p>"
};

using var cliente = new SmtpClient();
await cliente.ConnectAsync("smtp.proveedor.com", 587, MailKit.Security.SecureSocketOptions.StartTls);
await cliente.AuthenticateAsync("apikey", "TU_API_KEY");
await cliente.SendAsync(mensaje);
await cliente.DisconnectAsync(true);
```

**3. Encapsula el envío en un servicio inyectable**

No llames a MailKit desde un controlador. Define una interfaz y registra una implementación. Así puedes cambiar de proveedor o simular el envío en los tests.

```csharp
public interface IEmailSender
{
    Task EnviarAsync(string destinatario, string asunto, string cuerpoHtml);
}

// Program.cs
builder.Services.AddScoped<IEmailSender, MailKitEmailSender>();
```

```csharp
// En el caso de uso de "recuperar contraseña"
public class RecuperarPasswordHandler
{
    private readonly IEmailSender _email;

    public RecuperarPasswordHandler(IEmailSender email) => _email = email;

    public async Task EjecutarAsync(string correo, string enlaceReset)
    {
        var html = $"<p>Para restablecer tu contraseña entra <a href=\"{enlaceReset}\">aquí</a>.</p>";
        await _email.EnviarAsync(correo, "Recupera tu contraseña", html);
    }
}
```

**4. En desarrollo, no mandes correos reales**

Usa un servidor SMTP "de mentira" que captura los correos y te los enseña en una web local, en vez de enviarlos de verdad. Los más comunes son **Mailpit**, **MailHog** o **smtp4dev** (se levantan fácil con Docker). Apuntas tu configuración a ese SMTP local (`localhost`, puerto 1025 normalmente) y revisas los correos en su panel web.

```bash
docker run -d -p 1025:1025 -p 8025:8025 axllent/mailpit
# Configura Host=localhost, Port=1025 y abre http://localhost:8025 para ver los correos
```

**5. Verifica el dominio antes de producción**

Para que los correos lleguen bien, el cliente necesita un **dominio propio** (`miapp.com`) y configurar en su DNS los registros **SPF**, **DKIM** y **DMARC** que el proveedor te indique. Sin esto, aunque el envío "funcione", muchos correos irán a spam. El remitente debe usar ese dominio (`no-reply@miapp.com`), no un Gmail genérico.

**6. Envía fuera de la petición web si puedes**

Mandar un correo tarda y puede fallar. No bloquees la respuesta al usuario esperando al SMTP: idealmente encolas el envío (una cola en segundo plano, un *background service*, o el sistema de colas del proveedor) y reintenta si falla. Para empezar puede valer un envío directo, pero tenlo en mente al crecer.

## Lo que NO hace

- **No garantiza la entrega por sí solo** — necesitas dominio verificado y SPF/DKIM/DMARC bien configurados.
- **No sustituye a una herramienta de marketing** — el correo transaccional es uno-a-uno; las campañas masivas son otra cosa.
- **MailKit no es un proveedor** — es el *cliente* que habla SMTP; sigue necesitando un servidor (propio o de un proveedor) al que conectarse.
- **No deberías guardar las credenciales en el repositorio** — van en secretos, nunca en `appsettings.json` versionado.
- **No bloquees la experiencia del usuario** — un fallo al enviar no debería tumbar el registro o el reset; trátalo aparte.

---

*En resumen: el correo transaccional es el email automático que tu app manda a un usuario por una acción suya; en .NET se envía con MailKit hablando SMTP, pero la pieza que de verdad importa es tener un proveedor con buena reputación y el dominio bien configurado para que el correo llegue a la bandeja de entrada y no a spam.*
