# Correo transaccional — Guía de tecnología

Guía introductoria sobre cómo una aplicación envía correos automáticos (recuperar contraseña, confirmar registro, avisar de un pedido...) y cómo se hace en concreto desde .NET. Pensada para perfiles junior que nunca han montado el envío de email en una app y no saben por dónde empezar.

Aprenderás qué es el correo transaccional, por qué no basta con "asumir que hay un SMTP", cómo elegir proveedor y cómo enviar un correo desde C# con MailKit de forma sencilla y mantenible.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Correo transaccional en .NET](Correo-Transaccional.md) | Punto de partida completo: qué es, por qué la entregabilidad importa, cómo elegir proveedor y cómo enviar un correo con MailKit desde una app .NET. |

---

> ¿Vas a desplegar en la nube? El proveedor de correo suele depender de dónde corre la app (Amazon SES en AWS, Azure Communication Services en Azure). Cuando tengas claro el hosting, esta guía te ayuda a conectar las piezas.
