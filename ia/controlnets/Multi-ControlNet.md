# Multi-ControlNet

## ¿Qué es?

Multi-ControlNet es usar **varios [ControlNet](ControlNet.md) a la vez** sobre la misma generación: por ejemplo, un mapa de profundidad que fija el espacio y un esqueleto de pose que fija el gesto de la figura, actuando los dos simultáneamente sobre cada paso de denoising.

## ¿Por qué existe?

Porque cada [preprocesador](Preprocesadores.md) captura una sola dimensión de la estructura, y a veces la petición cruza dos.

El caso claro es el personaje dentro de una escena. Con **OpenPose** solo fijas el esqueleto: la pose sale clavada, pero el modelo coloca a la figura donde le parece, del tamaño que quiere y sin relación con el fondo. Con **depth** fijas el espacio y el volumen: la escena está bien, pero la pose se desdibuja porque el mapa de profundidad de una persona es una mancha suave que no dice dónde está cada articulación. Ninguno de los dos, por separado, expresa «esta persona, en esta pose, en este sitio de esta habitación».

Que se puedan combinar no es una funcionalidad añadida a posteriori: sale gratis de cómo funciona ControlNet. Como cada uno produce correcciones que se **suman** a las activaciones de la U-Net, y la suma es asociativa, dos ControlNet se combinan sumando también sus correcciones entre sí. No hay ninguna maquinaria de fusión: es una suma. El paper original ya lo contempla y muestra ejemplos de condicionamiento múltiple.

De esa simplicidad sale también su principal peligro, que es el tema de la segunda mitad de esta ficha: **nada impide sumar más de la cuenta**, y el sistema no avisa cuando lo haces.

## ¿Cuándo y para qué se usa?

Cuando hay dos restricciones que son de naturaleza distinta y las dos importan:

- **Un personaje en un decorado.** Pose por OpenPose, espacio por depth.
- **Un producto en una escena.** Contorno del producto por Canny, volumen del entorno por depth.
- **Interiorismo con muebles concretos.** Perspectiva por MLSD o depth, reparto del espacio por segmentación.
- **Escalar una ilustración conservando el trazo.** Contenido por tile, líneas por lineart.

Y cuándo **no**, que es la mitad importante de la respuesta:

- Cuando los dos mapas dicen lo mismo. Canny y lineart de la misma imagen no son dos restricciones, son una restricción aplicada dos veces.
- Cuando lo que quieres arreglar es una zona concreta. Para eso hay inpainting.
- Cuando aún no has calibrado bien el control que ya tenías. Apilar sobre un flujo mal ajustado multiplica los problemas en vez de resolverlos.

---

## Cómo se apila en `diffusers`

El cambio respecto a un solo control es mecánico: donde había un valor, ahora hay una **lista**. Y todas las listas tienen que ir en el mismo orden.

```python
import torch
from diffusers import StableDiffusionXLControlNetPipeline, ControlNetModel, AutoencoderKL

controlnets = [
    ControlNetModel.from_pretrained("diffusers/controlnet-depth-sdxl-1.0",
                                    torch_dtype=torch.float16),      # índice 0
    ControlNetModel.from_pretrained("xinsir/controlnet-openpose-sdxl-1.0",
                                    torch_dtype=torch.float16),      # índice 1
]

pipe = StableDiffusionXLControlNetPipeline.from_pretrained(
    "stabilityai/stable-diffusion-xl-base-1.0",
    controlnet=controlnets,                                          # una lista, no un modelo
    vae=AutoencoderKL.from_pretrained("madebyollin/sdxl-vae-fp16-fix",
                                      torch_dtype=torch.float16),
    torch_dtype=torch.float16,
    variant="fp16",
).to("cuda")
```

Y en la llamada:

```python
imagen = pipe(
    prompt="ilustración de personaje con abrigo de lana en un salón nórdico, luz de tarde",
    image=[mapa_profundidad, mapa_pose],              # mismo orden que la lista de modelos
    controlnet_conditioning_scale=[0.6, 0.8],         #    "        "
    control_guidance_start=[0.0, 0.0],                #    "        "
    control_guidance_end=[0.8, 0.6],                  #    "        "
    num_inference_steps=30,
    guidance_scale=7.0,
    generator=torch.Generator("cuda").manual_seed(42),
).images[0]
```

Ese alineamiento por posición es la fuente número uno de errores en esta ficha. El elemento 0 de `image` va al modelo 0, con el peso 0 y la ventana 0. Si intercambias dos mapas sin intercambiar sus pesos, el resultado sigue generándose sin ninguna queja — simplemente sale mal.

Una forma sencilla de que no pase es no escribir nunca las listas a mano:

```python
CONTROLES = [
    {"modelo": "diffusers/controlnet-depth-sdxl-1.0",  "mapa": mapa_profundidad, "peso": 0.6, "fin": 0.8},
    {"modelo": "xinsir/controlnet-openpose-sdxl-1.0",  "mapa": mapa_pose,        "peso": 0.8, "fin": 0.6},
]

imagen = pipe(
    prompt=PROMPT,
    image=[c["mapa"] for c in CONTROLES],
    controlnet_conditioning_scale=[c["peso"] for c in CONTROLES],
    control_guidance_end=[c["fin"] for c in CONTROLES],
    ...
).images[0]
```

Ahora cada control es una fila y no hay forma de descolocarlos. En cuanto pases de dos, esto deja de ser una manía y se vuelve necesario.

## Cómo se combinan de verdad: las correcciones se suman

Por dentro no hay nada más que esto. En cada paso de denoising:

1. Se ejecuta el ControlNet 0 con su mapa; produce sus correcciones y se multiplican por su peso.
2. Se ejecuta el ControlNet 1 con el suyo; lo mismo.
3. Las correcciones de los dos se **suman término a término**.
4. El resultado se suma a las activaciones de la U-Net, exactamente igual que con un solo control.

Tres consecuencias directas y muy prácticas:

- **El orden de la lista no cambia el resultado.** La suma es conmutativa. Lo único que importa del orden es que las cuatro listas estén alineadas entre sí. Si alguien te dice que hay que poner primero el «control principal», no es cierto — lo que importa es el peso, no la posición.
- **Los pesos se acumulan.** Dos controles a `0.8` no ejercen una presión de `0.8`: ejercen algo cercano a `1.6` sobre las mismas activaciones. Esto es lo que motiva la sección siguiente.
- **No hay negociación entre controles.** Si uno empuja hacia una silueta y el otro hacia otra distinta, el sistema no elige: suma las dos presiones y la U-Net recibe una corrección que no corresponde a ninguna imagen coherente. El resultado son duplicaciones y siluetas fantasma.

## El presupuesto de peso

De la acumulación sale la regla práctica más útil de la ficha:

> **Trata la suma de los pesos como un presupuesto de alrededor de 1.0 a 1.5, no como valores independientes.**

Con un solo control, `0.8` es un valor sensato. Con dos, poner `0.8` a cada uno duplica la presión y produce justo lo que se quería evitar: una imagen calcada, con el prompt ignorado y con artefactos en los bordes.

La forma de repartir el presupuesto es asignar roles, no repartir a partes iguales:

| Reparto | Ejemplo | Cuándo |
|---|---|---|
| **Dominante + apoyo** | `[0.8, 0.4]` | Lo habitual. Una restricción manda y la otra corrige. |
| **Equilibrado** | `[0.6, 0.6]` | Cuando las dos son igual de importantes y no se solapan. |
| **Dominante fuerte** | `[0.9, 0.3]` | Cuando la segunda es solo un empujón (p. ej. depth para dar volumen). |

Y una precisión importante: **el presupuesto no se reparte por igual entre mapas de densidad distinta**. Un mapa de pose lleva poquísima señal (unas rayas sobre negro) y un mapa de Canny lleva muchísima. En la práctica, `[canny 0.5, pose 0.9]` está más equilibrado que `[canny 0.7, pose 0.7]`, aunque los números digan lo contrario. La regla de la densidad de la ficha de [peso y ventana](Peso-y-Ventana-de-Control.md) sigue aplicando control por control.

## Ventanas desfasadas: repartir el turno en vez del volumen

Hay una segunda forma de que dos controles convivan, y está muy infrautilizada: en lugar de bajarles el peso a los dos, **hacer que actúen en momentos distintos**.

Tiene sentido porque los pasos del bucle no hacen lo mismo. Si un control se ocupa de la composición general y el otro del detalle, no necesitan estar activos a la vez:

```python
imagen = pipe(
    prompt="salón de estilo industrial, ladrillo visto, luz de tarde",
    image=[mapa_profundidad, mapa_canny],
    controlnet_conditioning_scale=[0.8, 0.6],
    control_guidance_start=[0.0, 0.3],       # depth desde el principio; canny entra después
    control_guidance_end=[0.5, 0.9],         # depth se retira pronto; canny se queda al detalle
).images[0]
```

Lo que hace ese reparto: en el primer tramo solo manda la profundidad, y establece el volumen y la perspectiva de la habitación. A partir del 30 % entra Canny y afina los contornos de los objetos, mientras la profundidad se retira al 50 %. En ningún momento hay dos controles a peso alto simultáneamente, así que la presión total nunca se dispara — pero el resultado tiene el espacio de uno y las líneas del otro.

Es la técnica que permite usar tres controles sin que la imagen se rompa. Y como efecto colateral, es más barata, porque cada ControlNet se ejecuta en menos pasos.

## Combinaciones que funcionan

Lo que tienen en común las buenas parejas: **cada control aporta un eje que el otro no tiene**.

| Combinación | Para qué | Pesos de partida |
|---|---|---|
| **Depth + OpenPose** | Un personaje situado en un espacio concreto. El clásico. | `[0.6, 0.8]` |
| **Canny + Depth** | Producto o arquitectura: contorno exacto más volumen creíble. Evita el aspecto plano de Canny solo. | `[0.5, 0.5]` |
| **Lineart + Depth** | Ilustración a render: el trazo manda, la profundidad da cuerpo. | `[0.7, 0.4]` |
| **Segmentación + Depth** | Interiorismo: qué hay en cada región y a qué distancia. Permite rediseñar la escena editando el mapa de segmentación a mano. | `[0.6, 0.6]` |
| **MLSD + Segmentación** | Arquitectura: la caja del espacio y el reparto de superficies, dejando el mobiliario libre. | `[0.7, 0.5]` |
| **Tile + Lineart** | Escalado de ilustración: tile mantiene el contenido, lineart evita que el trazo se difumine al ampliar. | `[0.9, 0.4]` |
| **OpenPose + Soft edge** | Personaje con silueta de vestuario definida, dejando libre el material de la ropa. | `[0.8, 0.4]` |

La pareja **depth + pose** merece un comentario porque es la que resuelve el problema que abría la ficha, y hay un detalle que la hace funcionar: la ventana de la pose se recorta pronto (`end` en 0.6-0.7). La articulación queda decidida en el primer tercio y, a partir de ahí, mantener el esqueleto activo solo entorpece el modelado del cuerpo y de la ropa.

## Combinaciones que se pelean

| Combinación | Por qué falla |
|---|---|
| **Canny + Lineart de la misma imagen** | Los dos describen bordes. No añades un eje, duplicas la presión sobre el que ya tenías. Sale un calco y el prompt deja de existir. |
| **Canny + Scribble de la misma imagen** | Igual, y encima se contradicen en el detalle: scribble simplifica lo que Canny detalla. |
| **Depth + Normal de la misma imagen** | Muy redundantes: las normales son, a grandes rasgos, la pendiente de la profundidad. Poca información nueva y mucha presión extra. |
| **Pose de una imagen + Depth de otra** | Geometrías incompatibles. El esqueleto pide un brazo donde el mapa de profundidad dice que hay pared. Aparecen miembros duplicados y siluetas fantasma. |
| **Segmentación + Canny, los dos a peso alto** | La segmentación quiere bloques limpios y Canny quiere contornos exactos; en las fronteras se contradicen y salen bordes sucios. Funciona solo con la segmentación baja. |
| **Tile + cualquier cosa a peso alto** | Tile ya impone la apariencia completa. Sumarle estructura encima no deja nada que decidir. |

El patrón detrás de todos ellos se resume en una pregunta que conviene hacerse antes de apilar:

> **¿Qué grado de libertad me quita el segundo control que no me quitara ya el primero?** Si no sabes contestarla, no lo apiles.

## Por qué apilar de más degrada

Con tres controles la cosa se pone difícil y con cuatro casi siempre está rota. Los motivos son cuatro y actúan a la vez:

**1. La corrección se sale del rango que la U-Net conoce.** Cada ControlNet se entrenó **solo**, sumando su corrección a un modelo base sin nada más. Cuando llegan tres correcciones sumadas, la U-Net recibe en sus capas intermedias magnitudes que nunca vio durante el entrenamiento. La respuesta a eso son artefactos: halos alrededor de los objetos, texturas planas, colores que se saturan.

**2. No quedan grados de libertad.** Una imagen tiene una cantidad finita de decisiones que tomar. Si la pose está fijada, el volumen está fijado, el contorno está fijado y el reparto por regiones está fijado, lo único que queda por decidir es el color de las superficies — y entonces el prompt se convierte en una paleta, no en una descripción. El síntoma es una imagen técnicamente correcta y completamente sin alma, y la gente suele achacárselo al modelo base cuando la culpa es del apilamiento.

**3. Las contradicciones no se resuelven, se promedian.** Dos mapas que piden cosas ligeramente distintas en el mismo sitio producen una corrección que no corresponde a ninguna de las dos. En imágenes eso se ve como duplicaciones, dedos de más, bordes dobles y objetos que empiezan de una forma y acaban de otra.

**4. El coste crece linealmente y sin compensación.** Este no es estético pero es el que mata en producción.

## El coste, que es lineal

Cada ControlNet activo es **una pasada completa de red por cada paso de denoising**. No hay reutilización entre ellos: son modelos distintos con pesos distintos.

| Controles | Coste de cómputo por imagen | VRAM adicional (SDXL, fp16) |
|---|---|---|
| 0 | Referencia | — |
| 1 | ≈ +30-40 % | ≈ 2,5 GB |
| 2 | ≈ +60-80 % | ≈ 5 GB |
| 3 | ≈ +90-120 % | ≈ 7,5 GB |

Con tres controles sobre SDXL puedes estar tardando el doble y ocupando 7-8 GB extra solo en pesos de ControlNet. En un cuaderno de pruebas da igual; en un servicio que genera bajo demanda, es la diferencia entre caber en una GPU y no caber.

Y aquí es donde las **ventanas desfasadas** dejan de ser una técnica de calidad para ser una de coste: un control con `end=0.5` se ejecuta la mitad de los pasos y cuesta la mitad. Un flujo de tres controles bien escalonados puede costar lo que uno y medio.

## Antes de apilar, mira si hay una alternativa

Apilar es la respuesta obvia y muchas veces no es la mejor. Cuatro alternativas que resuelven el mismo problema más barato:

- **Un ControlNet *union*.** Modelos como `xinsir/controlnet-union-sdxl-1.0` están entrenados para aceptar varios tipos de mapa con un único juego de pesos. Ahorran memoria (un modelo en lugar de tres) y, al haberse entrenado con las condiciones juntas, gestionan mejor su interacción que tres modelos que nunca se vieron. Si tu combinación está entre las que soporta, es preferible a apilar.
- **Editar el mapa a mano.** Muchas veces el segundo control existe para arreglar un defecto del primero. Si el mapa de Canny se come el fondo, píntalo de negro y ya no hace falta añadir profundidad para «recuperar el espacio». Un mapa es una imagen, y editarla cuesta menos que otra red corriendo treinta veces.
- **IP-Adapter para lo que no es geometría.** Si el segundo control lo querías para imponer paleta, ambiente o estilo, ningún ControlNet te lo va a dar bien: los mapas estructurales no llevan color. Un IP-Adapter con una imagen de referencia hace ese trabajo por otro canal —el de la atención cruzada— y no compite con el ControlNet por el mismo presupuesto.
- **Inpainting en dos fases.** Si la segunda restricción afecta solo a una parte de la imagen (la cara, un objeto concreto), generar primero con un control y luego regenerar esa zona con el otro suele dar mejor resultado que aplicar los dos a toda la imagen. Cada fase tiene su propio presupuesto entero.

## Errores frecuentes

- **Listas desalineadas.** El mapa de pose con el peso pensado para el de profundidad. No da error: da una imagen mala. Usa una lista de diccionarios.
- **Longitudes distintas.** Si pasas dos modelos y tres mapas, `diffusers` protesta con un `ValueError` sobre el número de imágenes y ControlNets; si pasas dos modelos y **un** peso escalar, en cambio, lo aplica a los dos sin avisar, y probablemente no es lo que querías.
- **Mezclar familias dentro de la lista.** Todos los ControlNet apilados tienen que ser de la familia del checkpoint. Uno de SD 1.5 colado entre dos de SDXL revienta con un error de dimensiones.
- **Mapas de tamaños distintos.** Todos deben tener la resolución y la proporción de la salida. Uno más pequeño se amplía y su estructura se desplaza respecto a los demás, lo que produce el desalineamiento clásico de contornos dobles.
- **Mantener los pesos de cuando había un solo control.** Al añadir el segundo hay que rebajar el primero. Es el error que más se repite.
- **Apilar antes de haber calibrado.** Si con un control la imagen ya no salía, con dos tampoco va a salir, y ahora hay el doble de variables.
- **Combinar dos mapas del mismo eje.** Canny más lineart no es redundancia inofensiva: es sobrecontrol.

## Buenas prácticas avanzadas

- **Calibra los controles de uno en uno, y en el orden en que mandan.** Ajusta el dominante a solas hasta que su parte del resultado esté bien; fíjalo; añade el segundo con peso bajo y súbelo hasta que aporte, rebajando el primero si hace falta. Calibrar dos a la vez es un espacio de búsqueda bidimensional donde no puedes atribuir nada a nada. La disciplina de un dial cada vez de la [ficha de peso y ventana](Peso-y-Ventana-de-Control.md) importa aquí el doble.
- **Escalona las ventanas antes de bajar los pesos.** Cuando dos controles se estorban, la reacción instintiva es rebajarlos a los dos, y eso debilita las dos restricciones a la vez. Repartirles el turno —uno en la primera mitad, otro en la segunda— conserva la fuerza de ambos y elimina la interferencia, porque nunca coinciden. Es la técnica que separa un apilamiento que funciona de uno que sobrevive a duras penas.
- **Aplica la prueba de la ablación antes de meterlo en producción.** Quita cada control por separado y vuelve a generar con la misma semilla. Si al quitar uno el resultado no empeora de forma visible, ese control **no está haciendo nada** y estás pagando un 30-40 % de cómputo por él. Es sorprendente la frecuencia con la que un flujo heredado arrastra un tercer control que dejó de aportar cuando alguien cambió el preprocesador del primero.
- **Prefiere un *union* a tres modelos cuando la memoria sea la restricción.** Tres ControlNet de SDXL en `float16` son unos 7,5 GB solo en pesos, encima del modelo base. Un modelo union cubre varias condiciones con un solo juego, y de paso las trata como se entrenaron: juntas.
- **Documenta la combinación como una receta con nombre.** Modelos, mapas, preprocesadores con sus parámetros, pesos y ventanas forman un conjunto que solo funciona completo: cambiar el umbral de Canny invalida su peso, y cambiar su peso desequilibra el presupuesto del otro. Guardarlo como «personaje-en-escena: depth 0.6 (0→0.8) + pose 0.8 (0→0.6), depth por Depth Anything V2» es reutilizable; guardar solo los pesos, no.

## Documentación oficial

- [ControlNet en la documentación de `diffusers`](https://huggingface.co/docs/diffusers/using-diffusers/controlnet) — la sección de MultiControlNet, con el ejemplo canónico de pose más Canny y el detalle de que todos los parámetros aceptan listas.
- [`StableDiffusionXLControlNetPipeline`](https://huggingface.co/docs/diffusers/api/pipelines/controlnet_sdxl) — la firma exacta: `controlnet_conditioning_scale`, `control_guidance_start` y `control_guidance_end` admiten `float` o `list[float]`, y ahí se ve qué se espera de cada uno.
- [*Adding Conditional Control to Text-to-Image Diffusion Models*](https://arxiv.org/abs/2302.05543) — el paper contempla el condicionamiento múltiple y muestra que basta con sumar las salidas de las ControlNet, sin ninguna interpolación ni ponderación especial.
- [`xinsir/controlnet-union-sdxl-1.0`](https://huggingface.co/xinsir/controlnet-union-sdxl-1.0) — la ficha del modelo *union* de referencia para SDXL: qué modos soporta y cómo se seleccionan.

## Recursos didácticos

- [Demo de ControlNet v1.1 en Hugging Face Spaces](https://huggingface.co/spaces/hysts/ControlNet-v1-1) — para generar rápido los mapas de dos preprocesadores y ver, antes de escribir nada, si describen ejes distintos o el mismo.
- [Wiki de `sd-webui-controlnet`](https://github.com/Mikubill/sd-webui-controlnet/wiki) — la sección de *Multi-ControlNet* explica el reparto de pesos desde la perspectiva de la interfaz, que es donde más gente lo ha usado y donde están documentadas las combinaciones que la comunidad ha ido decantando.
- [ComfyUI](https://github.com/comfyanonymous/ComfyUI) — encadenar dos nodos *Apply ControlNet* y ver el condicionamiento pasar de uno al siguiente es, con diferencia, la forma más rápida de entender que esto es una suma en cadena y no una fusión.

---

*En resumen: apilar ControlNet es sumar correcciones, así que los pesos se acumulan y el presupuesto es común — dos controles con roles distintos y turnos escalonados rinden mucho más que cuatro empujando a la vez.*
