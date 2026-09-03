# Peso y ventana de control

## ¿Qué es?

Son los dos diales que gobiernan un [ControlNet](ControlNet.md) una vez ya has elegido el mapa: **cuánto** manda la estructura (el peso) y **durante qué parte** del proceso de generación manda (la ventana de pasos). En `diffusers` son `controlnet_conditioning_scale` por un lado y la pareja `control_guidance_start` / `control_guidance_end` por otro.

## ¿Por qué existe?

Recuerda cómo actúa un ControlNet: en cada paso produce unos tensores de corrección que se **suman** a las activaciones internas de la U-Net. Una suma admite un multiplicador delante, y ese multiplicador es el peso. Nada obliga a que valga 1.

Que sea configurable no es un capricho de la API: es que **no existe un valor correcto universal**. Un mapa de pose son cuatro rayas sobre negro; un mapa de Canny de una foto con textura es una maraña de líneas que cubre la imagen entera. La misma cifra aplicada a los dos produce, en un caso, una pose apenas insinuada, y en el otro, un calco. La cantidad de señal que lleva el mapa cambia con el preprocesador, con la imagen y con la resolución, así que el peso hay que ponerlo cada vez.

La ventana existe por una razón distinta y más interesante. Como la imagen se construye [progresivamente](Modelos-de-Difusion.md#cómo-funciona-quitar-ruido-paso-a-paso), no todos los pasos hacen lo mismo: los primeros deciden dónde va cada cosa y los últimos deciden cómo es la superficie de las cosas. Imponer estructura tiene todo el sentido del mundo mientras se decide la composición, y bastante poco mientras se decide la textura de un tejido. La ventana permite decir «manda al principio y luego apártate», que es casi siempre lo que quieres.

> Si vienes de backend, el peso es un coeficiente y la ventana es un `WHERE` sobre el número de paso. Lo que cuesta no es entender los parámetros, es saber a cuál de los dos acudir cuando el resultado no es el que esperabas — y a eso va la mitad de esta ficha.

## ¿Cuándo y para qué se usa?

Siempre. No hay un flujo con ControlNet que no pase por aquí, porque el valor por defecto de `controlnet_conditioning_scale` es `1.0` y ese valor es **demasiado alto para casi todo**. La propia documentación de `diffusers`, en su ejemplo oficial de Canny con SDXL, usa `0.5` y lo comenta como «recomendado para buena generalización». Ese comentario es el resumen de esta ficha.

Los síntomas que traen a la gente aquí son tres:

- La imagen ignora el mapa y sale lo que le da la gana.
- La imagen es un calco plano del mapa, sin volumen ni materiales.
- La imagen respeta el mapa pero **ignora el prompt**: pediste terciopelo verde y sigue saliendo madera clara.

Los tres se arreglan con estos dos diales, y el tercero es el más contraintuitivo — la respuesta casi nunca es subir el CFG.

---

## Qué pasa en un paso, y por qué el «cuándo» importa

Antes de tocar nada, conviene tener presente qué se está decidiendo en cada momento del bucle de denoising. Con 30 pasos, a grandes rasgos:

| Tramo | Qué se está decidiendo | Cuánto importa la estructura |
|---|---|---|
| Pasos 1-8 (0-25 %) | Composición general, dónde va cada masa, silueta grande | Muchísimo |
| Pasos 9-20 (25-65 %) | Formas concretas, proporciones, relaciones entre objetos | Bastante |
| Pasos 21-30 (65-100 %) | Textura, materiales, grano, microdetalle | Casi nada |

Esa progresión no es una metáfora: es consecuencia de que el latente empieza siendo ruido y va perdiéndolo. Con mucho ruido solo se pueden decidir cosas grandes; con poco ruido ya solo queda ajustar lo pequeño.

De ahí sale la observación que rige todo lo demás: **la estructura se juega en el primer tercio**. Un ControlNet que actúa del paso 1 al 20 y luego se apaga produce una imagen con la geometría igual de respetada que si hubiera actuado hasta el 30, y con bastante más libertad de materiales. Y de propina, más rápida.

## El peso: `controlnet_conditioning_scale`

Multiplica las correcciones del ControlNet antes de sumarlas a la U-Net.

```python
imagen = pipe(
    prompt="silla tapizada en terciopelo verde esmeralda, patas de latón, fotografía de producto",
    image=mapa_bordes,
    controlnet_conditioning_scale=0.7,
    num_inference_steps=30,
    generator=torch.Generator("cuda").manual_seed(42),
).images[0]
```

Cómo se comporta el rango:

| Valor | Qué pasa |
|---|---|
| `0.0` | Sin control. Equivale a no haber puesto ControlNet. |
| `0.2-0.4` | Sugerencia. La composición general se parece; los detalles no. |
| `0.5-0.8` | **El rango de trabajo.** La estructura se respeta y el prompt sigue mandando en lo demás. |
| `0.9-1.0` | Obediencia estricta. Se nota en que el resultado empieza a parecer coloreado sobre un dibujo. |
| `> 1.0` | Sobrecontrol. Aparecen halos, contornos duros, texturas planas y colores raros. |

Ese último tramo merece una advertencia porque el parámetro no lo prohíbe. Valores como `1.5` o `2.0` amplifican una corrección que la U-Net no está preparada para recibir a esa escala: no obtienes «más fidelidad», obtienes artefactos. Si a `1.0` el mapa no se respeta, el problema está en el mapa o en el modelo, no en el multiplicador.

Y la regla que más ayuda a acertar a la primera:

> **Cuanto más denso es el mapa, más bajo debe ir el peso.** Un mapa disperso —una pose, un scribble de cuatro trazos— lleva poca señal y necesita `0.8-1.0` para hacerse notar. Un mapa denso —Canny sobre una foto con textura, o un tile— ya lleva señal de sobra en cada píxel, y con `0.9` ahoga al prompt.

## La ventana: `control_guidance_start` y `control_guidance_end`

Son dos fracciones entre 0 y 1 que dicen en qué **porcentaje del recorrido** empieza y acaba de aplicarse el control. Por defecto, `0.0` y `1.0`: todo el rato.

```python
imagen = pipe(
    prompt="silla tapizada en terciopelo verde esmeralda, fotografía de producto",
    image=mapa_bordes,
    controlnet_conditioning_scale=0.7,
    control_guidance_start=0.0,      # actúa desde el primer paso
    control_guidance_end=0.8,        # y se apaga al 80 % del recorrido
    num_inference_steps=30,
).images[0]
```

Con 30 pasos, ese `0.8` significa que la ControlNet corre en los pasos 1 a 24 y no se ejecuta en los seis últimos. Los seis últimos son justo los que ponen la textura del terciopelo.

### Recortar el final: casi siempre buena idea

Bajar `control_guidance_end` a `0.7-0.9` es la modificación con mejor relación entre beneficio y riesgo de toda la ficha, y produce tres efectos a la vez:

- **La estructura se conserva igual**, porque ya estaba decidida en los pasos que sí se ejecutaron.
- **Los materiales y la textura salen mucho mejor**, porque los últimos pasos trabajan sin la corrección tirando de ellos hacia el mapa.
- **Se ahorra tiempo**, porque esos pasos ejecutan una red menos.

El síntoma que te dice que lo necesitas es característico: una imagen con la geometría perfecta pero con aspecto de dibujo coloreado, con contornos demasiado nítidos y superficies sin grano. Eso es el ControlNet actuando en los pasos donde no pinta nada.

### Retrasar el inicio: rara vez, pero cuando toca, salva

Subir `control_guidance_start` por encima de `0` deja que el modelo decida solo la composición de los primeros pasos y solo después le impone el mapa. El resultado tiende a ser una imagen que se parece al mapa **de lejos** pero se ha tomado libertades en la disposición.

Es contraproducente casi siempre, porque es exactamente la información que querías imponer. Tiene un uso concreto: cuando el mapa y el prompt piden cosas incompatibles y quieres que gane el prompt en lo grande. Con un `start=0.2` sobre el mapa de la silla y un prompt de «sillón de orejas», sale un sillón de orejas cuya silueta recuerda a la silla, en lugar de una silla a la que le han pegado orejas.

### Las dos combinaciones que resumen todo

```python
# Estructura estricta, materiales libres — el patrón por defecto para producto
controlnet_conditioning_scale=0.8, control_guidance_start=0.0, control_guidance_end=0.8

# Estructura orientativa — cuando el mapa es solo una referencia de composición
controlnet_conditioning_scale=0.5, control_guidance_start=0.0, control_guidance_end=0.6
```

## Los modos de prioridad

Además del peso y la ventana hay una tercera palanca, menos conocida, que cambia **cómo** se reparte la influencia en lugar de cuánta hay. En `diffusers` se activa con `guess_mode`; en las interfaces gráficas aparece como un selector de tres opciones.

Para entenderlo hay que recordar dos detalles del funcionamiento interno:

1. Con guidance scale activo, cada paso hace **dos** predicciones: una con el prompt y otra con el negative. Normalmente el ControlNet corrige las dos.
2. Las correcciones no se inyectan en un solo sitio, sino en **trece puntos** distintos de la U-Net, de los más superficiales a los más profundos.

Los tres modos juegan con esas dos cosas:

| Modo en la interfaz | Qué hace | En `diffusers` |
|---|---|---|
| **Balanced** | El comportamiento normal: corrección completa en ambas ramas y en los trece puntos. | Por defecto |
| **My prompt is more important** | Escala las correcciones de forma descendente entre los trece puntos, de modo que las capas profundas reciben mucho menos. La composición se mantiene y el prompt recupera terreno en el detalle. | Se aproxima bajando el peso y recortando `control_guidance_end` |
| **ControlNet is more important** | El ControlNet solo corrige la rama del prompt, no la del negative, y con la misma rampa descendente. | `guess_mode=True` |

`guess_mode` tiene un uso muy concreto y bastante llamativo: **generar sin prompt**. Con el modo activo, la ControlNet intenta reconocer por sí sola qué hay en el mapa y guiar la generación hacia ello:

```python
imagen = pipe(
    prompt="",                       # sin prompt, a propósito
    image=mapa_profundidad,
    guess_mode=True,
    guidance_scale=4.0,              # entre 3 y 5, como recomienda la documentación
    controlnet_conditioning_scale=1.0,
).images[0]
```

Ese `guidance_scale` bajo no es opcional: sin prompt que amplificar, los valores habituales de 7-8 producen imágenes quemadas. Es un modo para explorar qué «ve» el ControlNet en un mapa, más que para producción.

## El protocolo de calibración

Aquí está el método, y es simple hasta el aburrimiento. Lo que lo hace valioso es que casi nadie lo sigue, y sin él lo que se hace es probar números y quedarse con el que salió bonito, sin aprender nada transferible a la imagen siguiente.

**Uno. Fija todo lo que no estés midiendo.** Semilla, prompt, negative, pasos, guidance scale, scheduler y mapa. La semilla, en particular, con un `Generator` nuevo en cada llamada:

```python
def generar(peso, fin=1.0, semilla=42):
    return pipe(
        prompt=PROMPT,
        negative_prompt=NEGATIVE,
        image=MAPA,
        controlnet_conditioning_scale=peso,
        control_guidance_end=fin,
        num_inference_steps=30,
        guidance_scale=7.0,
        generator=torch.Generator("cuda").manual_seed(semilla),   # nuevo cada vez
    ).images[0]
```

**Dos. Usa un scheduler determinista.** Los ancestrales reinyectan ruido y no convergen, así que dos ejecuciones con parámetros distintos se diferencian también por el azar. `DPMSolverMultistepScheduler` o `UniPCMultistepScheduler`.

**Tres. Mueve un dial cada vez, en barrido grueso.** Cinco valores separados, no diez juntos:

```python
for peso in [0.3, 0.5, 0.7, 0.9, 1.1]:
    generar(peso).save(f"peso-{peso}.png")
```

**Cuatro. Mira las cinco juntas, no de una en una.** El ojo compara mucho mejor que recuerda:

```python
from PIL import Image

pesos = [0.3, 0.5, 0.7, 0.9, 1.1]
imgs = [generar(p) for p in pesos]

rejilla = Image.new("RGB", (1024 * len(imgs), 1024))
for i, img in enumerate(imgs):
    rejilla.paste(img, (1024 * i, 0))
rejilla.save("barrido-peso.png")
```

**Cinco. Afina en fino alrededor del ganador.** Si `0.7` fue el mejor, prueba `0.6`, `0.65`, `0.75`, `0.8`.

**Seis. Solo entonces, pasa al dial siguiente.** Con el peso ya fijado, barre `control_guidance_end` en `[0.5, 0.7, 0.9, 1.0]`.

**Siete. Valida con tres semillas más.** Un valor que solo funciona con la semilla 42 no es un valor calibrado, es una casualidad. Repite el ganador con tres semillas distintas antes de darlo por bueno.

Y el error que invalida todo el protocolo, por si acaso: **cambiar el prompt a media calibración**. Si a mitad del barrido decides que el prompt quedaba mejor de otra forma, el barrido ha terminado. Vuelve a empezar.

## Puntos de partida por preprocesador

Con lo anterior no necesitas esta tabla, pero acorta la primera iteración. Son puntos de partida para SDXL, no verdades:

| Preprocesador | Peso | `end` | Por qué |
|---|---|---|---|
| Canny | 0.5-0.7 | 0.8 | Mapa denso y duro: se pasa de fiel enseguida. |
| Soft edge (HED, PiDiNet) | 0.6-0.8 | 0.8 | Al ser gradual, tolera algo más de peso. |
| Lineart | 0.6-0.8 | 0.8 | Similar a soft edge. |
| Scribble | 0.8-1.0 | 0.9 | Mapa muy pobre en señal: necesita peso para notarse. |
| MLSD | 0.6-0.8 | 0.7 | Solo hay rectas; apagarlo pronto libera el mobiliario. |
| Depth | 0.5-0.7 | 0.8 | Con peso alto aplana la iluminación. |
| Normal | 0.6-0.8 | 0.8 | Marca mucho el relieve; vigila el sobrecontrol. |
| OpenPose | 0.8-1.0 | 0.6-0.7 | Mapa dispersísimo. Peso alto y apagado temprano: la pose ya está decidida. |
| Segmentación | 0.6-0.8 | 0.8 | Regiones grandes; peso medio para que no salgan bloques planos. |
| Tile | 0.8-1.0 | 1.0 | Aquí sí quieres fidelidad hasta el final: es su trabajo. |

La lógica de la columna del peso es la regla de la densidad: pose y scribble arriba, Canny y depth abajo.

## Diagnóstico: el síntoma y el dial

La tabla que resuelve el 90 % de los casos:

| Lo que ves | Causa probable | Qué tocar |
|---|---|---|
| No respeta el mapa en absoluto | Mapa vacío, familia equivocada, o le estás pasando la foto en vez del mapa | Mira el mapa **antes** de tocar el peso |
| Respeta poco, se parece de lejos | Peso bajo | Peso ↑ en pasos de 0.2 |
| Calco plano, sin volumen ni materiales | Peso alto o mapa demasiado denso | Peso ↓; si no basta, preprocesador más laxo |
| Geometría perfecta, textura de dibujo coloreado | El control sigue activo en los pasos finales | `control_guidance_end` a 0.7-0.8 |
| El prompt se ignora (pediste verde, sale marrón) | El mapa lleva información que el prompt contradice | Peso ↓, **y** revisa que el mapa sea acromático |
| Halos, contornos duros, colores quemados | Peso por encima de 1.0 | Peso ≤ 1.0 |
| Composición general rara aunque el mapa sea bueno | `control_guidance_start` > 0 | `start` a 0.0 |
| Todo correcto pero soso y sin detalle fino | Ventana demasiado larga | `end` a 0.6-0.8 |

Fíjate en la fila del prompt ignorado, porque es la que más tiempo hace perder. La reacción instintiva es subir `guidance_scale`, y eso quema los colores sin resolver nada: el prompt no se ignora por falta de fuerza, se ignora porque hay otra señal empujando en contra. Se baja la otra señal, no se sube esta.

## Los mismos diales en otras interfaces

| `diffusers` | AUTOMATIC1111 | ComfyUI | APIs de proveedor |
|---|---|---|---|
| `controlnet_conditioning_scale` | *Control Weight* | `strength` en *Apply ControlNet* | `weight`, o una etiqueta tipo `Low`/`Mid`/`High` |
| `control_guidance_start` | *Starting Control Step* | `start_percent` | Rara vez expuesto |
| `control_guidance_end` | *Ending Control Step* | `end_percent` | Rara vez expuesto |
| `guess_mode` | *Control Mode* | Nodo aparte | No expuesto |

Que la ventana casi nunca esté disponible en las APIs de proveedor tiene una consecuencia práctica: **allí solo tienes el peso**, y por tanto no puedes resolver el caso «geometría perfecta pero textura de dibujo» de la forma buena. La alternativa es bajar el peso, que también relaja la geometría. Si te encuentras peleando con eso contra una API, el problema no es tu calibración: es que falta un mando.

Y un aviso al traducir código entre pipelines de `diffusers`: en los pipelines de **image-to-image** e **inpainting** con ControlNet, el mapa **no** va en `image` sino en `control_image`, porque `image` ya está ocupado por la imagen de partida. Es un error silencioso muy fácil de cometer al copiar un ejemplo de text-to-image.

## Errores frecuentes

- **Dejar el peso por defecto.** `1.0` es el valor de la API, no una recomendación. Empieza por 0.6-0.7.
- **Reutilizar el mismo `Generator` en varias llamadas.** Se consume, así que la segunda imagen sale con otra semilla y el barrido no compara nada. Créalo dentro de la función de generación.
- **Calibrar con un scheduler ancestral.** El azar reinyectado en cada paso enmascara el efecto del parámetro.
- **Mover peso y ventana a la vez.** Si mejora, no sabes por cuál; si empeora, tampoco.
- **Aceptar un valor validado con una sola semilla.** Prueba tres antes de fijarlo en producción.
- **Confundir `guidance_scale` con `controlnet_conditioning_scale`.** El primero es la fuerza del prompt, el segundo la del mapa. Se llaman parecido y hacen cosas opuestas.
- **Subir el peso porque «no se parece» cuando el mapa está vacío.** El mapa negro de un OpenPose que no detectó a nadie no mejora con peso 2.0.

## Buenas prácticas avanzadas

- **Recorta `control_guidance_end` antes de bajar el peso.** Son dos formas de aflojar, pero no equivalentes: bajar el peso afloja la geometría **también al principio**, que es justo donde la querías firme; recortar el final la deja intacta y solo libera la textura. Cuando el síntoma es «materiales pobres» y no «geometría demasiado rígida», el dial correcto es la ventana. Casi todo el mundo empieza por el peso y acaba sacrificando composición sin necesidad.
- **Calibra sobre la imagen más difícil de tu lote, no sobre la más fácil.** El valor que funciona en el mockup limpio de fondo blanco se queda corto en la foto con textura, mientras que el que funciona en la difícil suele valer también para la fácil. Calibrar con el caso amable garantiza tener que recalibrar.
- **Guarda la tripleta ganadora junto al preprocesador, no junto a la imagen.** Peso, ventana y umbrales del preprocesador van juntos: cambiar los umbrales de Canny de `100/200` a `50/100` densifica el mapa y **invalida el peso calibrado**. Una tabla de perfiles («producto-canny: umbrales 100/200, peso 0.7, end 0.8») es reutilizable; una nota pegada a una imagen concreta, no.
- **Usa la ventana como optimización de coste, no solo de calidad.** Con `end=0.7` y 30 pasos, la ControlNet se ejecuta 21 veces en lugar de 30: ahorras casi un tercio de su sobrecoste con una pérdida de fidelidad geométrica que en la mayoría de los casos no se ve. En un servicio que genera miles de imágenes, esa cifra es dinero, y es la optimización que menos se aplica porque nadie asocia el parámetro con el gasto.
- **Ante un resultado incoherente, sospecha del mapa antes que del dial.** El orden de depuración correcto es: mirar el mapa, comprobar la familia del ControlNet, comprobar que va en el parámetro correcto (`image` o `control_image`), y **solo entonces** tocar números. La mayoría de las sesiones largas de ajuste de peso son en realidad un mapa mal generado que nadie abrió.

## Documentación oficial

- [`StableDiffusionXLControlNetPipeline` en la referencia de `diffusers`](https://huggingface.co/docs/diffusers/api/pipelines/controlnet_sdxl) — la definición exacta de `controlnet_conditioning_scale`, `control_guidance_start`, `control_guidance_end` y `guess_mode`, con sus valores por defecto y sus tipos (que admiten listas, para varios controles).
- [*Adding Conditional Control to Text-to-Image Diffusion Models*](https://arxiv.org/abs/2302.05543) — la sección de método explica por qué la corrección se suma y en qué puntos, que es lo que da sentido a que exista un multiplicador.
- [Wiki de `sd-webui-controlnet`](https://github.com/Mikubill/sd-webui-controlnet/wiki) — la documentación de los *Control Mode* y de cómo se implementan las rampas de peso entre capas.

## Recursos didácticos

- [Demo de ControlNet v1.1 en Hugging Face Spaces](https://huggingface.co/spaces/hysts/ControlNet-v1-1) — permite mover el peso y ver el efecto sin montar nada. Es la forma más rápida de calibrar la intuición.
- [Diffusion Explainer](https://poloclub.github.io/diffusion-explainer/) — para ver, paso a paso, cómo la composición queda decidida al principio y la textura al final. Es la observación de la que sale toda la lógica de la ventana.
- [Documentación de X/Y/Z plot en AUTOMATIC1111](https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Features#xyz-plot) — el barrido en rejilla con interfaz, si prefieres no escribir el bucle. La idea es la misma que el script de esta ficha.

---

*En resumen: el peso decide cuánto manda la estructura y la ventana decide hasta cuándo — y como la composición se juega en los primeros pasos, apagar el control antes del final casi siempre da la misma geometría con mejores materiales y menos tiempo.*
