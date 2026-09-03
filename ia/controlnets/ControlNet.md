# ControlNet

## ¿Qué es?

ControlNet es una red adicional que se acopla a un modelo de difusión ya entrenado y le impone **dónde** va cada cosa. Le pasas un mapa —los bordes de una silla, la profundidad de un salón, el esqueleto de un personaje— y el modelo genera una imagen nueva, con el estilo y los materiales que pida el prompt, pero respetando esa geometría.

> Esta ficha da por conocido el vocabulario básico de la difusión: latente, U-Net, paso de denoising, guidance scale, checkpoint, image-to-image. Si alguno de esos términos no te dice nada, empieza por [Modelos de difusión](Modelos-de-Difusion.md) y vuelve; son quince minutos y el resto de la colección se lee sin tropiezos. El vocabulario propio de ControlNet sí se define entero aquí.

## ¿Por qué existe?

Un modelo de difusión acepta una sola vía de entrada: el texto. Y el texto es pésimo describiendo geometría.

Prueba a explicar con palabras la forma exacta de una silla concreta —el ángulo del respaldo, la curva de los reposabrazos, la proporción entre asiento y patas— con la precisión suficiente para que salga *esa* silla y no otra parecida. No se puede. El lenguaje sirve para «silla escandinava de madera clara», no para «esta silla».

Antes de ControlNet, la única forma de meter una imagen en el proceso era [image-to-image](Modelos-de-Difusion.md#image-to-image-y-su-strength), y eso trae otro problema que la siguiente sección desmonta en detalle. La aportación de ControlNet, publicada en 2023 por Lvmin Zhang, Anyi Rao y Maneesh Agrawala en *Adding Conditional Control to Text-to-Image Diffusion Models*, es abrir **un segundo canal de entrada** al modelo: uno por el que entra solo estructura, sin arrastrar color ni textura, y que se puede regular por separado del prompt.

El paper ganó el premio al mejor artículo de ICCV 2023, y con razón: en unos meses pasó de ser una técnica a ser la forma normal de usar estos modelos en cualquier contexto profesional. La razón es sencilla — sin control geométrico, un generador de imágenes es una máquina tragaperras; con él, es una herramienta de producción.

## ¿Cuándo y para qué se usa?

Siempre que la petición tenga la forma «una imagen nueva **que respete esto**»:

- **Recolorear un mockup de producto conservando su forma.** Diez acabados de la misma silla, y que sea reconociblemente la misma silla en las diez.
- **Variaciones de una ilustración manteniendo la pose de un personaje.** Cambia el vestuario, el fondo y el estilo; el gesto y la posición de los miembros no.
- **Convertir un boceto en render.** El dibujo a lápiz manda la composición; el modelo pone materiales, luz y acabado.
- **Mantener la composición de una foto de interiorismo cambiando el estilo.** El mismo salón, la misma perspectiva y los mismos muebles en su sitio, pero de estilo industrial en vez de nórdico.
- **Coherencia entre imágenes de una serie.** Un catálogo entero generado sobre el mismo esqueleto de composición.

En todos ellos hay una parte que es sagrada (la geometría) y otra que es libre (todo lo demás). ControlNet existe justo para poder decir cuál es cuál.

---

## ControlNet no es image-to-image

Esta es la confusión más común y merece la pena resolverla antes de seguir, porque explica por qué mucha gente pasa horas moviendo un parámetro que nunca les va a dar lo que buscan.

En **image-to-image**, la imagen de entrada se codifica a latente, se le añade ruido y el bucle arranca desde ahí. Toda la información de la entrada —forma, color, textura, iluminación, fondo— viaja por el mismo sitio y se regula con un único dial, `strength`.

```python
# ❌ MAL — intentar "misma silla, otro material" con image-to-image
imagen = pipe_img2img(
    prompt="silla tapizada en terciopelo verde esmeralda, fotografía de producto",
    image=mockup_silla,          # la silla es de madera clara
    strength=0.5,
).images[0]
```

Con `strength=0.5` la salida sale a medio camino de todo: la forma se conserva a medias y el color se ha movido a medias. No hay ningún valor de `strength` que dé el resultado pedido, y no por falta de puntería:

| `strength` | Forma | Material |
|---|---|---|
| 0.2 | Idéntica | Sigue siendo madera clara |
| 0.5 | Reconocible pero deformada | Un verde apagado sobre madera |
| 0.8 | Ya no es la misma silla | Terciopelo verde perfecto |

El dial no puede separar estructura de apariencia **porque no las tiene separadas**. Van juntas en el mismo latente.

En **ControlNet**, la imagen de entrada no es el punto de partida del bucle: el bucle sigue empezando desde ruido puro, como en text-to-image. Lo que se hace con la imagen es extraerle un mapa de estructura y meterlo por un canal aparte:

```python
# ✅ BIEN — la estructura por su canal, el material por el prompt
mapa_bordes = detector_canny(mockup_silla)      # solo la silueta, sin color

imagen = pipe_controlnet(
    prompt="silla tapizada en terciopelo verde esmeralda, fotografía de producto",
    image=mapa_bordes,                          # la geometría, y solo la geometría
    controlnet_conditioning_scale=0.7,          # cuánto se respeta esa geometría
).images[0]
```

El mapa de bordes es blanco y negro: **no contiene la madera clara**. Esa información se ha tirado a propósito, así que el prompt tiene el campo libre para poner terciopelo verde sin pelearse con nada.

La diferencia en una tabla:

| | image-to-image | ControlNet |
|---|---|---|
| De dónde parte el bucle | Del latente de tu imagen, con ruido | De ruido puro |
| Qué información de la entrada llega | Toda: forma, color, textura, luz | Solo la que capture el mapa |
| Cuántos diales | Uno (`strength`) | Peso, ventana de pasos, y uno por cada control apilado |
| Qué puede pedirse | «Algo parecido a esto» | «Esta estructura exacta, todo lo demás nuevo» |

No son alternativas rivales: son herramientas para peticiones distintas, y se pueden combinar (image-to-image con ControlNet encima es un patrón habitual). Pero si lo que quieres es «esta forma, otra apariencia», `strength` no es el sitio donde buscar.

## Cómo condiciona la generación: un canal aparte

Recuerda el bucle de denoising: en cada paso, la U-Net mira el latente ruidoso, el prompt y el número de paso, y predice el ruido.

Con ControlNet, en cada paso ocurre además esto:

1. La red ControlNet recibe **lo mismo** que la U-Net (latente ruidoso, prompt, paso) **más** el mapa de condicionamiento.
2. Produce una colección de tensores de corrección, uno por cada nivel de la U-Net.
3. Esos tensores se **suman** a lo que la U-Net iba a pasar por sus *skip connections* y por el cuello de botella.
4. La U-Net continúa como si nada, pero con sus activaciones internas desviadas hacia la estructura que marca el mapa.

Tres consecuencias que conviene interiorizar ya:

- **Es una suma, no una sustitución.** Por eso hay un peso que multiplica esa corrección antes de sumarla, y por eso se pueden apilar varios ControlNet: las correcciones se suman entre sí. Es el tema de [Peso y ventana de control](Peso-y-Ventana-de-Control.md) y [Multi-ControlNet](Multi-ControlNet.md).
- **Ocurre en cada paso.** La ControlNet no se ejecuta una vez: se ejecuta tantas veces como pasos de denoising haya, porque su entrada incluye el latente ruidoso del momento. De ahí que tenga un coste apreciable —del orden de un tercio más de tiempo por imagen— y que tenga sentido apagarla a partir de cierto paso.
- **El modelo base no se modifica.** Sus pesos están congelados. Puedes usar el mismo ControlNet con cualquier checkpoint de la misma familia: el de fotorrealismo, el de ilustración, el que sea.

> El contraste con **T2I-Adapter**, una alternativa más ligera, aclara este último punto: los adapters calculan sus características **una sola vez** a partir del mapa y las reutilizan en todos los pasos, porque no reciben el latente ruidoso. Son mucho más baratos y bastante menos precisos. ControlNet paga el coste de mirar el estado actual en cada paso, y eso es lo que le permite corregir con precisión.

## La arquitectura: copia entrenable y zero convolutions

Aquí está la idea que hace que todo esto funcione, y es sorprendentemente elegante.

**El problema de partida.** Quieres enseñarle una capacidad nueva a un modelo que costó millones entrenar, y solo dispones de unas decenas de miles de ejemplos de esa capacidad. Un *fine-tuning* normal con tan pocos datos produce *catastrophic forgetting*: el modelo aprende tu tarea y de paso destroza todo lo que sabía. Empeora tanto que deja de merecer la pena.

**La solución de ControlNet**, en tres piezas:

1. **El modelo base se congela entero.** Sus pesos no se tocan ni una vez durante el entrenamiento. Lo que sabía, lo conserva por construcción.

2. **Se hace una copia entrenable de los bloques del encoder de la U-Net** (los doce bloques descendentes y el bloque central). Esa copia arranca con los pesos del original —o sea, no parte de cero: hereda toda la comprensión visual del modelo base— y es la que se entrena. El decoder no se copia: no hace falta, porque la inyección se hace en las *skip connections* que van del encoder al decoder.

3. **Las dos mitades se conectan mediante *zero convolutions*:** convoluciones 1×1 con **peso y sesgo inicializados a cero**. Hay una a la entrada de la copia (por donde entra el mapa de condicionamiento) y una a la salida de cada bloque, justo antes de sumarse al modelo congelado.

**Por qué las zero convolutions son la clave.** Una convolución 1×1 con todo a cero produce ceros salga lo que salga. Eso significa que **en el paso 0 del entrenamiento, la aportación de ControlNet es literalmente nula**:

```
salida_del_conjunto = U-Net_congelada(x) + 0  ≡  U-Net_congelada(x)
```

El modelo compuesto es, exactamente y bit a bit, el modelo base. No hay ni un gramo de ruido aleatorio entrando en las capas profundas de una red que costó una fortuna. Compáralo con lo que pasa si esas conexiones se inicializan al azar, como es habitual: durante las primeras miles de iteraciones el modelo escupe basura y va aprendiendo a la vez a controlar *y* a reparar el daño que él mismo se hizo. Con datasets pequeños nunca termina de recuperarse.

La objeción evidente es: si todo es cero, ¿no se queda ahí atascado para siempre? No, y el motivo es que la derivada respecto a los **pesos** de una capa lineal es su **entrada**, no su peso. Con entrada distinta de cero, el gradiente respecto a los pesos no es cero, así que en la primera actualización dejan de valer cero y la capa empieza a transmitir. El punto cero es un punto de partida, no un punto fijo.

Esto tiene dos consecuencias muy prácticas:

- **Se puede entrenar un ControlNet con pocos datos y hardware modesto.** El paper original muestra resultados útiles con del orden de decenas de miles de pares imagen-condición, y el sobrecoste frente a entrenar el modelo solo es de aproximadamente un 23 % más de memoria y un 34 % más de tiempo por iteración. Es la diferencia entre «esto lo hace un laboratorio» y «esto lo hace cualquiera con una GPU buena».
- **El entrenamiento no converge suavemente: da un salto.** Es el llamado *sudden convergence phenomenon*. Durante miles de pasos la salida ignora el condicionamiento y de golpe, en unas pocas iteraciones, empieza a obedecerlo. Si alguna vez entrenas uno, no lo abandones antes de tiempo pensando que no aprende.

Todo esto explica por qué existen decenas de ControlNet distintos publicados por gente muy diversa: entrenar uno nuevo para un tipo de condicionamiento propio está al alcance de un equipo pequeño.

## Preprocesador y modelo ControlNet: dos piezas distintas

Esta distinción es la fuente número uno de resultados desconcertantes, así que conviene fijarla con nombre y apellidos.

- El **preprocesador** (también llamado *annotator*, «anotador») es un programa —a veces un algoritmo clásico, a veces una red pequeña— que convierte una **imagen normal** en un **mapa de condicionamiento**. `cv2.Canny` es un preprocesador. Un estimador de profundidad es un preprocesador. No sabe nada de difusión.
- El **modelo ControlNet** es la red entrenada que **consume** ese mapa y lo traduce en correcciones para la U-Net. Cada modelo está entrenado para un tipo concreto de mapa: el de Canny espera bordes blancos sobre negro y no entiende un mapa de profundidad.

```
foto de la silla  →  [preprocesador Canny]  →  mapa de bordes  →  [ControlNet canny]  →  correcciones
     imagen                  algoritmo            imagen B/N          red entrenada
```

**El error clásico** es pasar por el preprocesador algo que ya es un mapa. Si tienes un dibujo de líneas hecho a mano y lo pasas por el detector de bordes, lo que obtienes son *los bordes de las líneas*: cada trazo se convierte en dos trazos paralelos, uno a cada lado. El ControlNet los respeta obedientemente y la imagen sale con todo doblado.

```python
# ❌ MAL — el boceto ya es un mapa de líneas
mapa = detector_canny(boceto_a_lapiz)     # dos líneas por cada trazo
imagen = pipe(prompt=..., image=mapa)

# ✅ BIEN — un mapa ya listo va directo al modelo
mapa = boceto_a_lapiz.convert("RGB")      # sin preprocesar
imagen = pipe(prompt=..., image=mapa)
```

Este reparto tiene una implicación importante y poco intuitiva: **`diffusers` no preprocesa nada por ti**. El parámetro `image` de un pipeline de ControlNet espera el mapa ya hecho. Las interfaces gráficas sí lo hacen, y por eso quien viene de ellas suele pasar la foto original y no entender por qué el resultado es un desastre. En [Preprocesadores](Preprocesadores.md) están todos, uno a uno.

## Instalación y primer ejemplo de punta a punta

Sobre la instalación de la ficha anterior, hacen falta dos cosas más:

```bash
pip install controlnet_aux opencv-python
```

- `controlnet_aux` — la colección de preprocesadores empaquetada por Hugging Face.
- `opencv-python` — necesaria para Canny, que es un algoritmo clásico de visión por computador y no una red.

El ejemplo completo: partimos del mockup de una silla de madera clara y queremos la misma silla tapizada en terciopelo verde.

**Paso 1 — extraer el mapa de bordes.**

```python
import cv2
import numpy as np
from PIL import Image
from diffusers.utils import load_image

original = load_image("mockup-silla.png").resize((1024, 1024))

bordes = cv2.Canny(np.array(original), 100, 200)   # umbrales bajo y alto
bordes = np.stack([bordes] * 3, axis=-1)           # de 1 canal a 3, que es lo que espera el modelo
mapa = Image.fromarray(bordes)
mapa.save("mapa-bordes.png")                       # míralo siempre antes de generar
```

Ese `mapa-bordes.png` es una imagen en blanco y negro con la silueta de la silla y sus líneas interiores. **Ábrelo.** Es el hábito que más tiempo ahorra: el 80 % de los resultados raros se ven venir mirando el mapa.

**Paso 2 — cargar el ControlNet y el pipeline.**

```python
import torch
from diffusers import StableDiffusionXLControlNetPipeline, ControlNetModel, AutoencoderKL

controlnet = ControlNetModel.from_pretrained(
    "diffusers/controlnet-canny-sdxl-1.0",         # ← entrenado para SDXL, no vale para SD 1.5
    torch_dtype=torch.float16,
)

vae = AutoencoderKL.from_pretrained(               # evita el problema del VAE de SDXL en fp16
    "madebyollin/sdxl-vae-fp16-fix", torch_dtype=torch.float16,
)

pipe = StableDiffusionXLControlNetPipeline.from_pretrained(
    "stabilityai/stable-diffusion-xl-base-1.0",
    controlnet=controlnet,
    vae=vae,
    torch_dtype=torch.float16,
    variant="fp16",
).to("cuda")
```

El pipeline es el de siempre con un argumento nuevo: `controlnet`. Todo lo demás —U-Net, VAE, codificadores— es idéntico, y eso es exactamente lo que promete la arquitectura de copia congelada.

**Paso 3 — generar.**

```python
imagen = pipe(
    prompt="silla tapizada en terciopelo verde esmeralda, patas de latón, "
           "fotografía de producto, fondo blanco, luz de estudio",
    negative_prompt="borroso, deformado, marca de agua, texto",
    image=mapa,                              # el mapa, NO la foto original
    controlnet_conditioning_scale=0.7,       # cuánto manda la estructura
    num_inference_steps=30,
    guidance_scale=7.0,
    generator=torch.Generator("cuda").manual_seed(42),
).images[0]

imagen.save("silla-terciopelo.png")
```

El resultado es la misma silla —mismo respaldo, mismas proporciones, mismo ángulo— en terciopelo verde con patas de latón. Cambia el prompt dejando todo lo demás igual y tendrás la variante siguiente del catálogo, con la geometría clavada.

El parámetro nuevo es `controlnet_conditioning_scale`, que a 0.7 significa «respeta la estructura pero deja algo de margen». Es el primero de los diales que se calibran, y tiene su propia ficha: [Peso y ventana de control](Peso-y-Ventana-de-Control.md).

## Elegir el ControlNet que corresponde a tu checkpoint

**Los ControlNet de SD 1.5 y los de SDXL no son intercambiables.** Ni un poco. No es cuestión de calidad: son arquitecturas distintas, con distinto número de bloques y distintas dimensiones internas, y la copia entrenable de una no encaja en la otra.

Lo que pasa al mezclarlos tiene dos formas, y la peligrosa es la segunda:

```python
# ❌ MAL — ControlNet de SD 1.5 sobre un pipeline SDXL
controlnet = ControlNetModel.from_pretrained("lllyasviel/control_v11p_sd15_canny", ...)
pipe = StableDiffusionXLControlNetPipeline.from_pretrained("stabilityai/...", controlnet=controlnet, ...)
```

```
RuntimeError: mat1 and mat2 shapes cannot be multiplied (2x2048 and 768x320)
```

Ese error de dimensiones es el caso amable: te enteras enseguida. El caso malo es cuando las formas casualmente encajan lo bastante para no reventar y lo que sale son imágenes ruidosas, sucias o vagamente relacionadas con el mapa. Ahí se pierden horas culpando al peso, al prompt o al preprocesador.

La regla, entonces:

| Familia del checkpoint | ControlNet que le corresponde | Ejemplos de identificadores |
|---|---|---|
| SD 1.5 y sus derivados | Los `control_v11*_sd15_*` | `lllyasviel/control_v11p_sd15_canny` |
| SDXL 1.0 y sus derivados | Los que digan `sdxl` en el nombre | `diffusers/controlnet-canny-sdxl-1.0`, `xinsir/controlnet-union-sdxl-1.0` |
| Flux.1 | Los específicos de Flux | `InstantX/FLUX.1-dev-Controlnet-Canny` |

Dos apuntes sobre esos nombres, que parecen crípticos y no lo son:

- En la nomenclatura oficial de SD 1.5, `control_v11**p**` significa que el modelo está listo para producción, `control_v11**e**` que es experimental y `control_v11**u**` que está sin terminar. El `f1` de `control_v11f1p_sd15_depth` indica que es una corrección de errores sobre la versión anterior. Merece la pena preferir los `p`.
- «Familia» incluye los checkpoints afinados por la comunidad. Un modelo de ilustración construido sobre SDXL admite cualquier ControlNet de SDXL, aunque su autor no haya hecho nada al respecto. Eso es precisamente lo que compra congelar el modelo base.

## La resolución del mapa de condicionamiento

El mapa y la imagen que vas a generar tienen que ser **coherentes en tamaño y en proporción**. `diffusers` redimensiona el mapa a la resolución de salida si no coincide, y ahí es donde aparecen los problemas silenciosos:

- **Proporción distinta** → el mapa se deforma. Un mapa cuadrado estirado a 1216×832 da una silla achatada, y el modelo la reproduce achatada muy fielmente.
- **Mapa mucho más pequeño que la salida** → al ampliarlo, las líneas se vuelven gruesas y borrosas, y el ControlNet interpreta ese engrosamiento como parte de la estructura.
- **Mapa mucho más grande** → al reducirlo se pierden las líneas finas, que simplemente desaparecen.

La forma sencilla de no equivocarse es preparar el mapa exactamente al tamaño de salida:

```python
ANCHO, ALTO = 1024, 1024

original = load_image("mockup-silla.png").resize((ANCHO, ALTO))
mapa = Image.fromarray(np.stack([cv2.Canny(np.array(original), 100, 200)] * 3, axis=-1))

imagen = pipe(prompt=..., image=mapa, width=ANCHO, height=ALTO, ...).images[0]
```

Y hay una segunda resolución, distinta de esta, que también importa: **la resolución a la que trabaja el preprocesador**. Muchos preprocesadores basados en redes (profundidad, pose, soft edge) analizan la imagen a un tamaño interno fijo y devuelven el mapa a ese tamaño. Si su tamaño interno es 512 y luego amplías a 1024, pierdes detalle que sí estaba en el original. Las interfaces gráficas tienen una opción llamada *Pixel Perfect* que calcula esa resolución interna para que coincida con la de salida; en `diffusers` se controla pasando `detect_resolution` e `image_resolution` al detector.

Y recuerda las resoluciones nativas: 512×512 para SD 1.5, 1024×1024 para SDXL. Un ControlNet impecable no salva a un modelo generando fuera de su rango.

## Lo que un mapa de bordes no puede transmitir: el color

Un mapa de Canny, de lineart o de scribble es **acromático**: literalmente blanco sobre negro. No hay ningún sitio donde guardar el color, así que la información cromática del original se ha destruido por completo antes de llegar al modelo.

Esto no es una limitación: es exactamente el mecanismo que hace útil la técnica.

- **Sirve perfectamente para «misma estructura, otra paleta».** La silla de madera clara y la silla de terciopelo verde producen mapas de bordes idénticos. Por eso el prompt puede reasignar el color sin resistencia.
- **No sirve en absoluto para «misma paleta, otra estructura».** Si lo que quieres es «genérame algo nuevo pero con estos colores y este ambiente», ningún mapa de bordes te va a llevar ahí, porque el color ni siquiera entra en el sistema.

Para ese segundo caso existe otra familia de herramientas: las **referencias de estilo**, que funcionan por un mecanismo distinto —típicamente **IP-Adapter**, que codifica la imagen de referencia con un codificador de imagen y la inyecta por las capas de atención cruzada, el mismo sitio por donde entra el prompt—. Ahí la imagen actúa como «prompt visual» y transmite paleta, textura y ambiente, pero no impone geometría. Es la simetría exacta de ControlNet, y las dos se combinan bien: estructura por ControlNet, paleta por IP-Adapter.

Un matiz para no sobregeneralizar: no todos los mapas son acromáticos. Los de **profundidad** son en escala de grises, donde el gris significa distancia, no luminosidad. Los de **segmentación** sí son de colores, pero cada color es la etiqueta de una clase («pared», «suelo», «silla») según una paleta fija; el azul de un mapa de segmentación no significa que ahí haya algo azul. Ninguno transmite la apariencia del original.

## Lo mismo visto desde otro sitio: ComfyUI, A1111 y las APIs de proveedor

Todo lo anterior es una sola cosa vista en `diffusers`. En la práctica te vas a encontrar ControlNet con tres caras muy distintas, y reconocer que son la misma pieza por debajo es lo que permite traducir un ajuste de un sitio a otro.

### ComfyUI: el grafo

ComfyUI expone el pipeline como un grafo de nodos, y ahí ControlNet se ve casi tal cual es. Los nodos relevantes son tres:

- **Load ControlNet Model** — carga el `.safetensors` del modelo. Equivale a `ControlNetModel.from_pretrained(...)`.
- Un nodo **preprocesador** (`Canny`, `Depth Anything`, `OpenPose Pose`...), que viene de un paquete de nodos aparte. Equivale al `detector`. En el grafo se ve literalmente que es una caja **anterior** y separada.
- **Apply ControlNet** — toma el condicionamiento de texto, el modelo ControlNet y el mapa, y devuelve un condicionamiento modificado. Sus mandos son `strength`, `start_percent` y `end_percent`.

Lo bueno del grafo es que la relación «preprocesador → mapa → modelo» es visible: los cables la dibujan. Y apilar dos ControlNet es encadenar dos nodos *Apply*, que es también lo que ocurre por dentro.

### AUTOMATIC1111: la pestaña

En la interfaz web de A1111, ControlNet es una extensión con un panel plegable debajo del prompt. Sus campos, y su traducción:

| Campo en la interfaz | Equivalente en `diffusers` |
|---|---|
| *Preprocessor* | El detector que aplicas antes (`CannyDetector`, `MidasDetector`...) |
| *Model* | `ControlNetModel.from_pretrained(...)` |
| *Control Weight* | `controlnet_conditioning_scale` |
| *Starting / Ending Control Step* | `control_guidance_start` / `control_guidance_end` |
| *Control Mode* | Los modos de prioridad, incluido `guess_mode` |
| *Resize Mode* | Cómo se ajusta el mapa a la proporción de salida |
| *Pixel Perfect* | Cálculo automático de la resolución del preprocesador |

Aquí el preprocesador y el modelo son dos desplegables **contiguos**, y la interfaz los ejecuta en cadena por ti. Es cómodo y es también donde nace la confusión de creer que son una sola cosa: en cuanto pasas a `diffusers` y solo hay un parámetro `image`, hay que saber que el primer desplegable ya no lo hace nadie.

### Las APIs de proveedor: un array de parámetros

Cuando ControlNet llega a través de la API de un proveedor de generación de imágenes, se reduce a unas cuantas claves en un JSON. La forma varía según el proveedor, pero el esqueleto es siempre reconocible:

```json
{
  "prompt": "silla tapizada en terciopelo verde esmeralda, fotografía de producto",
  "modelId": "<id del modelo base>",
  "controlnets": [
    {
      "initImageId": "a3f9c1e2-...",   // la imagen que subiste previamente
      "preprocessorId": 19,            // un número de catálogo: "bordes", "profundidad"...
      "weight": 0.7,                   // el peso, otra vez
      "strengthType": "Mid"            // a veces el peso viene como etiqueta, no como número
    }
  ]
}
```

Y hay proveedores que ni siquiera lo llaman ControlNet: exponen endpoints tipo `/control/structure` o `/control/sketch` con un único parámetro `control_strength`, y el tipo de control lo determina la ruta.

Vale la pena detenerse en lo que este formato **oculta**:

- El `preprocessorId` es un número de un catálogo propietario. No dice qué algoritmo hay detrás, ni con qué parámetros corre, ni a qué resolución. Dos proveedores pueden llamar «bordes» a cosas distintas.
- **El proveedor ejecuta el preprocesador por ti**, así que le mandas la foto original, no el mapa. Es lo contrario de `diffusers`, y es una fuente de confusión al migrar entre los dos.
- No hay ni rastro de la copia entrenable, de las zero convolutions ni de la familia del modelo base. Esa última omisión es la que muerde: si el proveedor cambia el modelo base por defecto de una familia a otra, los `preprocessorId` disponibles cambian con él, y una llamada que funcionaba deja de hacerlo con un mensaje poco informativo.
- Rara vez se expone la **ventana de pasos**. Si el resultado sale demasiado pegado al mapa, ahí solo tienes el peso para arreglarlo.

Mucha gente usa ControlNet exclusivamente por esta vía y nunca llega a saber que debajo hay una red neuronal entrenada aparte que corre en paralelo a la U-Net en cada paso. Se puede trabajar así, pero cuando algo va mal —y va a ir mal— la diferencia entre depurarlo y probar números al azar es exactamente lo que has leído en esta ficha.

## Errores frecuentes

- **La imagen ignora completamente el mapa.** Revisa por este orden: que estés pasando el mapa y no la foto original; que el ControlNet sea de la familia de tu checkpoint; y que `controlnet_conditioning_scale` no esté a un valor ridículo. Si el mapa está casi vacío (Canny con umbrales muy altos sobre una foto de poco contraste), no hay nada que obedecer.
- **La imagen es un calco plano del mapa, sin volumen ni materiales.** El peso está demasiado alto o el mapa demasiado denso. Baja a 0.5-0.6 o recorta la ventana de control.
- **Todo sale con las líneas dobladas o con un contorno fantasma alrededor.** Has preprocesado un mapa que ya era un mapa.
- **`RuntimeError: mat1 and mat2 shapes cannot be multiplied`** — ControlNet de una familia, checkpoint de otra.
- **Aparecen artefactos en los bordes de la imagen.** Suele ser el mapa redimensionado con una proporción distinta y rellenado. Prepara el mapa al tamaño exacto de salida.
- **`ValueError: Image size must be divisible by 8`** — el latente mide un octavo de la imagen. Redondea `width` y `height` a múltiplos de 8 (y preferiblemente de 64).

## Buenas prácticas avanzadas

- **Guarda y mira siempre el mapa antes de generar.** Suena obvio y casi nadie lo hace. El mapa es la única prueba de qué está viendo realmente el modelo, y los fallos más caros —un mapa vacío, uno invertido, uno con la proporción cambiada— se detectan de un vistazo y son invisibles mirando solo el resultado. Un `mapa.save(...)` en el pipeline de producción cuesta milisegundos y ahorra sesiones enteras.
- **Fija la semilla antes de tocar el peso, y muévelo de uno en uno.** Con la semilla suelta, cada prueba cambia dos cosas y no puedes atribuir la mejora a ninguna. Este protocolo es tan central que tiene su propia sección en [Peso y ventana de control](Peso-y-Ventana-de-Control.md).
- **Elige el preprocesador por lo que quieres que sea libre, no por lo que quieres conservar.** Es el cambio de mentalidad que separa a quien usa esto bien. Si el material debe poder cambiar, necesitas un mapa que no lleve material; si la iluminación debe poder cambiar, no uses profundidad, que la insinúa. La pregunta correcta no es «¿qué captura mejor esta imagen?» sino «¿qué es lo mínimo que tengo que imponer?».
- **Comprueba de dónde vienen los pesos del ControlNet que cargas.** Hay muchos publicados por la comunidad, de calidad muy desigual, y algunos están entrenados sobre un checkpoint afinado concreto en vez de sobre el base: funcionan bien con ese y regular con los demás. Ante un comportamiento raro, prueba primero con el ControlNet de referencia de la familia antes de dar por hecho que el problema es tuyo.
- **Vigila el coste antes de meterlo en producción.** Cada ControlNet activo es una pasada adicional de red **por cada paso de denoising**, y se nota en latencia y en VRAM. Recortar la ventana de control (que la ControlNet solo corra en el 60 % inicial de los pasos) suele mantener el resultado y devolver una parte del tiempo — es la optimización más rentable y la que menos se aplica.

## Documentación oficial

- [*Adding Conditional Control to Text-to-Image Diffusion Models*](https://arxiv.org/abs/2302.05543) (Zhang, Rao y Agrawala, 2023) — el paper original. La sección de método explica las zero convolutions y los experimentos de tamaño de dataset justifican por qué esto funciona con pocos datos.
- [ControlNet en la documentación de `diffusers`](https://huggingface.co/docs/diffusers/using-diffusers/controlnet) — la guía práctica de la librería, con los pipelines de text-to-image, image-to-image e inpainting con control.
- [Repositorio `lllyasviel/ControlNet-v1-1-nightly`](https://github.com/lllyasviel/ControlNet-v1-1-nightly) — la referencia del autor sobre los modelos v1.1: qué hace cada uno, qué preprocesador le corresponde y qué significan los sufijos del nombre.

## Recursos didácticos

- [ControlNet 1.1 en el wiki de `sd-webui-controlnet`](https://github.com/Mikubill/sd-webui-controlnet/wiki/ControlNet-1.1) — un catálogo con ejemplos visuales de entrada y salida para cada modelo. Es la forma más rápida de hacerse una idea de qué hace cada uno sin generar nada.
- [Demo de ControlNet en Hugging Face Spaces](https://huggingface.co/spaces/hysts/ControlNet-v1-1) — se prueba en el navegador, sin instalar nada. Sube una foto, cambia el preprocesador y observa cómo cambia el mapa; el aprendizaje va más rápido viéndolo que leyéndolo.
- [*The Illustrated Stable Diffusion*](https://jalammar.github.io/illustrated-stable-diffusion/) — si la parte de la U-Net y las *skip connections* no ha terminado de asentarse, aquí está dibujada.

---

*En resumen: ControlNet abre un segundo canal de entrada al modelo por el que entra solo geometría — y como el modelo base sigue congelado detrás de unas convoluciones inicializadas a cero, ese canal se añade sin estropear nada de lo que ya sabía.*
