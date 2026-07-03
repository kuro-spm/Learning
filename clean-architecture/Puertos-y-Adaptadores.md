# Puertos y Adaptadores

**¿Qué es?** Una forma de nombrar las fronteras de la aplicación: un **puerto** es una interfaz que define *qué* necesita o *qué* ofrece el centro; un **adaptador** es la pieza concreta que conecta ese puerto con una tecnología real del exterior. Es el vocabulario de la "arquitectura hexagonal", muy emparentada con Clean Architecture.

---

## ¿Por qué existe?

Cuando el centro de la aplicación define una interfaz, conviene tener un nombre para esos huecos por donde entra y sale la información. La metáfora del puerto viene de un ordenador: el centro tiene "puertos" (como un USB) con una forma fija, y al exterior se le pide un "adaptador" que encaje en ese puerto y hable con la tecnología real.

Así se separan dos preguntas: *qué* necesita la aplicación (el puerto, estable) y *cómo* se satisface (el adaptador, reemplazable).

---

## ¿Cuándo y para qué se usa?

Aparece siempre que la aplicación se comunica con el exterior, en dos direcciones:

- **Puertos de entrada (driving):** por donde *llega* una petición al centro. Una API REST o una pantalla son adaptadores de entrada que invocan un caso de uso.
- **Puertos de salida (driven):** por donde el centro *pide* algo a fuera. Un repositorio o una pasarela de pago son adaptadores de salida.

Por ejemplo, en una tienda online el puerto de salida `IRepositorioPedidos` puede tener un adaptador para SQL Server y otro de prueba en memoria, sin que el centro note la diferencia.

---

## Lo mínimo que necesitas saber

**1. El puerto es la interfaz (vive en el centro)**

```csharp
// Puerto de salida: el centro declara qué necesita.
public interface INotificador
{
    Task Enviar(string destinatario, string mensaje);
}
```

**2. El adaptador es la implementación (vive en el borde)**

```csharp
// Adaptador de salida: conecta el puerto con una tecnología real.
public class NotificadorEmail : INotificador
{
    public Task Enviar(string destinatario, string mensaje) { /* SMTP real */ }
}
```

**3. Un mismo puerto admite varios adaptadores**

Email, SMS o un adaptador falso para tests cumplen el mismo puerto `INotificador`. El centro funciona igual con cualquiera; solo cambia cuál conectas al arrancar.

---

## Lo que NO hace

- **No es una tecnología distinta de Clean Architecture** — es otro vocabulario para la misma idea de aislar el centro tras interfaces.
- **No exige una forma "hexagonal"** literal — el hexágono es solo un dibujo; lo que importa es la separación puerto/adaptador.
- **No añade reglas nuevas de negocio** — un adaptador solo traduce; las reglas siguen en el dominio.

---

## Buenas prácticas avanzadas

- **Nombra el puerto por la necesidad, no por la tecnología** — un puerto llamado `IEmailSender` en el centro ya es una decisión técnica colada de contrabando; `INotificador` deja abierta la puerta a email, SMS o push sin tocar nada. Si el nombre del puerto delata cuál es el adaptador, la frontera está cortada por el sitio equivocado.
- **Pasa la misma suite de tests a todos los adaptadores de un puerto** — el adaptador SQL y el falso en memoria deben comportarse igual en los bordes (¿qué devuelve si el pedido no existe?, ¿qué pasa al guardar un duplicado?). Una suite de *contract tests* compartida que se ejecuta contra ambos evita el clásico "los tests pasan con el fake y producción falla con el real".
- **La entrada también se adapta** — es fácil vigilar los puertos de salida y olvidar que un controlador REST es un adaptador: su único trabajo es traducir HTTP (rutas, códigos de estado, JSON) a llamadas al caso de uso. Si tipos del framework web cruzan hacia el centro, has roto la frontera por la puerta de entrada.

---

*En resumen: el puerto es el hueco con forma fija que define el centro, y el adaptador es la pieza intercambiable que lo conecta con la tecnología real — el mismo puerto, muchos adaptadores posibles.*
