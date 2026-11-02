# Historial de la consulta a la IAG — Actividad de verificación (Lab 4)

**Herramienta:** Claude (Anthropic), en una sesión nueva sin acceso al notebook ni al enunciado del laboratorio.
**Condiciones:** no se anexaron imágenes ni se mencionó el nombre del conjunto de datos; solo se describieron las imágenes.

---

## Mensaje 1 (estudiante)

Estoy haciendo un proyecto de segmentación de lesiones de piel sin aprendizaje supervisado. Tengo fotos a color tomadas con un dermatoscopio: cada una muestra una lesión pigmentada (de marrón oscuro a marrón claro, a veces con zonas rosadas o rojizas) más o menos centrada sobre piel clara. La lesión ocupa entre el 3 % y el 40 % de la imagen. Algunas fotos tienen pelos, pecas alrededor, reflejos de luz y esquinas negras por el viñeteado del dermatoscopio. Represento cada píxel como [β·R, β·G, β·B, λ·x, λ·y], con el color y las coordenadas normalizados a [0, 1], y aplico K-medias. ¿Qué valores concretos de β, λ y K me recomiendas? ¿Qué criterio automático usarías para decidir cuáles grupos son la lesión y construir una máscara binaria?

## Respuesta 1 (IAG)

Antes de dar valores, una observación: en K-medias solo importa la relación λ/β, no los valores absolutos. Si multiplicas β y λ por la misma constante, cambian todas las distancias en la misma proporción y obtienes la misma partición. Por eso conviene fijar β = 1 y ajustar solo λ.

**Valores recomendados**

- **β = 1.**
- **λ ≈ 0,25.** Un rango razonable es de 0,15 a 0,35.
  - Con color en [0, 1], la diferencia entre lesión y piel suele ser de unos 0,25 a 0,45 por canal. Eso da una distancia en RGB de aproximadamente 0,4 a 0,7.
  - Dentro de una lesión, las distancias espaciales van de 0,1 (lesión del 3 %) a 0,35 (lesión del 40 %). Multiplicadas por λ = 0,25 quedan entre 0,03 y 0,09. Así el color decide la partición, y la posición solo suaviza el ruido y ayuda a separar las esquinas oscuras de la lesión central.
  - Con λ ≥ 0,7 a 1, K-medias empieza a cortar la imagen en regiones espaciales, como un diagrama de Voronoi, y la lesión se fragmenta o se mezcla con piel.
  - Con λ = 0 pierdes la única pista que separa las esquinas negras y las pecas de la lesión, porque tienen un color parecido.
- **K = 4.** También puedes probar K = 5 si hay mucho pelo o muchos reflejos. Los grupos que esperas son:
  - la lesión oscura o núcleo,
  - la lesión clara o zonas rosadas,
  - la piel,
  - las esquinas negras, el pelo o los reflejos.

  Con K = 2, las esquinas negras y la lesión suelen caer en el mismo grupo. Con K > 6, la piel se divide por gradientes de iluminación.

**Preprocesamiento que más ayuda**

1. Reduce la imagen a unos 256 px de lado. K-medias será más rápido.
2. Quita las esquinas por viñeteado. Marca como "fondo" los píxeles con luminancia < 0,15 que estén conectados al borde, o usa una máscara circular. Luego exclúyelos del clustering.
3. Elimina el pelo. Aplica un filtro *black-hat* morfológico con un elemento lineal o elíptico de unos 11 a 17 px. Umbraliza el resultado y rellena con `cv2.inpaint`.
4. Atenúa los reflejos. Satura o haz *inpainting* de los píxeles con luminancia > 0,95.
5. Aplica un suavizado gaussiano con σ ≈ 1,5 a 2 px.
6. Ejecuta K-medias con `n_init=10` y una semilla fija.

**Criterio automático para elegir los grupos que son lesión**

Para cada grupo k calcula tres medidas:

- **L̄ₖ:** la luminancia media, por ejemplo 0,299R + 0,587G + 0,114B o L* de Lab.
- **dₖ:** la distancia de su centroide espacial al centro (0,5; 0,5).
- **bₖ:** la fracción de sus píxeles que tocan una franja del 5 % junto al borde de la imagen.

Con esas medidas:

1. **Descarta** los grupos con bₖ > 0,3. Son piel periférica o esquinas.
2. Elige como **semilla** el grupo con menor puntuación sₖ = L̄ₖ + 0,5·dₖ, es decir, el más oscuro y central.
3. **Añade** cualquier otro grupo con L̄ₖ < L̄_piel − 0,10 y dₖ < 0,3. Así recuperas las zonas claras o rosadas de la lesión. Aquí L̄_piel es la luminancia del grupo más grande que toca el borde. Si las zonas rosadas quedan fuera, usa también el canal a* de Lab: incluye los grupos con a* mayor que el de la piel en al menos 8 unidades.
4. **Postprocesa** la máscara:
   - Aplica una apertura y luego un cierre morfológicos con un disco de radio 3 a 5 px.
   - Conserva solo la componente conexa más grande que contenga el centro o esté más cerca de él. Esto elimina las pecas.
   - Rellena los huecos con `binary_fill_holes`.
5. **Controla la cordura del resultado.** Si el área final queda fuera del rango del 2 % al 50 %, repite con K + 1 o relaja el umbral.

---

## Mensaje 2 (estudiante)

Si K lo tengo que elegir automáticamente con el coeficiente de silueta entre 2 y 8, ¿cambia tu recomendación de β y λ?

## Respuesta 2 (IAG)

Sí, cambia un poco, por estas razones.

1. **La silueta se calcula en el mismo espacio de características.** Por eso depende de λ/β, igual que K-medias. Con λ alto, las coordenadas (x, y) se distribuyen de forma uniforme y no forman grupos naturales. Entonces la silueta queda baja y casi igual para todos los K (alrededor de 0,3 a 0,4) y elige K casi al azar, a menudo uno alto. Por eso conviene **bajar λ a unos 0,1 a 0,15, con β = 1**. Así la silueta mide sobre todo la estructura de color, que es la que tiene grupos reales.

2. **La silueta tiende a favorecer K = 2 o 3.** Si dejas las esquinas negras dentro, la separación "esquinas frente a resto" es tan fuerte que la silueta elegirá K = 2 y la lesión puede quedar mezclada con la piel. Por eso es casi obligatorio **excluir antes el viñeteado, el pelo y los reflejos**. Una alternativa es calcular la silueta solo con las columnas de color, aunque el clustering use también (x, y). Esa variante suele dar elecciones de K más estables.

3. **Las lesiones pequeñas (3 %) aportan pocos puntos a la silueta.** La silueta pesa a todos los píxeles por igual, así que puede preferir dividir la piel en dos tonos antes que aislar una lesión pequeña. Dos soluciones:
   - Restringe el rango a K de 2 a 6. Rara vez necesitarás más.
   - En caso de empate (diferencia < 0,02), elige el K menor.

4. **Hay un detalle práctico de coste.** La silueta es O(n²), así que calcúlala sobre una submuestra de 5 000 a 10 000 píxeles. En sklearn puedes usar `silhouette_score(X, labels, sample_size=8000, random_state=0)`.

5. **El criterio de selección de grupos debe funcionar con cualquier K.** Por eso conviene mantener el criterio de la respuesta 1: descartar los grupos que tocan el borde y luego juntar todos los grupos oscuros y centrales. Con K = 2 obtendrás un solo grupo de lesión. Con K = 5 a 6 se unirán varios. En ambos casos, el postprocesado morfológico y la componente conexa central siguen igual.

En resumen: usa β = 1 y λ ≈ 0,1 a 0,15. Calcula la silueta sin las esquinas ni el pelo, sobre una submuestra, con K entre 2 y 6 y desempatando hacia el K menor.
