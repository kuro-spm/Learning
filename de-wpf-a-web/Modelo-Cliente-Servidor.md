# Modelo Cliente-Servidor

## ¿Qué es?

Es la forma en la que está organizada casi toda la web: el programa se parte en dos
mitades que viven en máquinas distintas. El **cliente** (el navegador en el ordenador
de quien lo usa) pide cosas, y el **servidor** (tu programa, en otra máquina) las
prepara y responde.

## ¿Por qué existe?

Una app de WPF es **monolítica**: la interfaz, la lógica y a menudo hasta los datos
viven en el mismo proceso, en el mismo ordenador. Tú arrastras un botón en el
diseñador, escribes el `Click` en el code-behind y todo ocurre ahí mismo, en memoria.

En la web eso no es posible: la persona que usa tu aplicación está en otro ordenador,
quizá al otro lado del mundo, con un navegador que tú no controlas. Por eso el programa
se divide: el navegador muestra la interfaz, y tu código (el servidor) hace el trabajo
y guarda los datos. Se comunican enviándose mensajes por la red.

> Si vienes de WPF, piensa en el cliente-servidor como separar tu `MainWindow` (que
> pasa a ser HTML en el navegador de otra persona) de tu lógica de negocio y tus
> repositorios (que se quedan en el servidor). Ya no comparten memoria: hablan por la red.

## ¿Cuándo y para qué se usa?

Es el modelo por defecto de cualquier aplicación web. Una tienda online es el ejemplo
clásico: el navegador de la clienta muestra el catálogo (cliente), pero el precio, el
stock y el pago se calculan y guardan en el servidor. Lo mismo con un blog, una app de
tareas o un panel de gestión: la pantalla está en un sitio, los datos y las reglas en otro.

## Lo mínimo que necesitas saber

**1. El cliente y el servidor no comparten memoria**

En WPF tu `ViewModel` y tus datos están en el mismo proceso: lees una propiedad y ya
está. En web, el cliente solo tiene lo que el servidor le haya enviado. Para conseguir
algo nuevo, tiene que **pedirlo** por la red.

```text
Navegador (cliente)  ──── "dame los productos" ───►  Servidor (tu C#)
Navegador (cliente)  ◄──── lista de productos ─────  Servidor (tu C#)
```

**2. El servidor es quien manda con los datos y las reglas**

Nunca confíes en el cliente: el navegador puede ser manipulado por quien lo usa. Las
validaciones importantes, los precios y los permisos se comprueban **siempre** en el
servidor.

**3. Un servidor atiende a muchos clientes a la vez**

Tu app de WPF tenía una sola usuaria: la persona delante de la pantalla. Un servidor
web atiende a cientos de clientes simultáneos. Por eso no puedes guardar el estado de
una persona en una variable global "de la app" como harías en escritorio.

**4. La interfaz se describe con HTML, no con XAML**

El navegador no entiende XAML. La interfaz viaja como HTML (la estructura) y CSS (el
aspecto). Tu servidor genera o sirve ese HTML para que el navegador lo dibuje.

## Lo que NO hace

- **No comparte objetos entre cliente y servidor** — solo intercambian mensajes (texto, normalmente JSON o HTML).
- **No garantiza que el cliente se comporte bien** — la seguridad real está en el servidor.
- **No mantiene una conexión abierta permanente** por defecto — cada petición es independiente (ver [HTTP](HTTP.md)).

## Buenas prácticas avanzadas

- **Diseña asumiendo que la red fallará** — en WPF una llamada a un método o funcionaba o lanzaba excepción; en la red hay un tercer estado: no saber qué pasó (enviaste la petición y nunca llegó respuesta). Los diseños que aguantan en producción ponen *timeouts* a toda llamada remota y piensan qué ocurre si el cliente reintenta: ¿se crearía el pedido dos veces? Ese "¿y si se repite?" debe responderse en el servidor, no confiarse a que la red siempre funcione.
- **Trata cada viaje de ida y vuelta como un coste de diseño** — entre cliente y servidor hay decenas o cientos de milisegundos por viaje, y no se optimizan con mejor código sino con *menos viajes*. El error típico es trasladar el estilo de escritorio (muchas llamadas pequeñas, como quien lee propiedades en memoria) a la web: una pantalla que hace 15 peticiones para pintarse será lenta aunque cada una tarde poco. Agrupa: una petición que traiga todo lo que la pantalla necesita.
- **Valida dos veces, confía una** — la validación en el cliente existe solo para la experiencia de uso (avisar al instante); la del servidor es la única real, porque cualquiera puede fabricar peticiones a mano saltándose tu interfaz. Quien domina la web duplica las validaciones sin sentirse culpable: son dos responsabilidades distintas, no código repetido.
- **El "estado global" cambia de casa: de variables a base de datos o caché** — en producción suele haber varias copias del servidor detrás de un balanceador, y dos peticiones de la misma persona pueden caer en máquinas distintas. Cualquier dato guardado en una variable estática o en memoria del proceso desaparece o se desincroniza. Si un dato debe sobrevivir a la petición, vive en una base de datos o una caché compartida, nunca "en la app".
- **Protege al servidor de sus clientes** — un servidor atiende a desconocidos: pon límites a lo que aceptas (tamaño de peticiones, ritmo de llamadas, valores de entrada) igual que un local pone aforo. En escritorio el peor cliente eras tú misma; en la web puede ser un script malicioso haciendo mil peticiones por segundo.

---

*En resumen: en la web tu programa deja de ser una sola pieza en un ordenador y pasa a ser dos mitades —navegador y servidor— que se hablan por la red; el servidor es quien guarda los datos y pone las reglas.*
