# Laboratorio 4 - Segmentación de imágenes con técnicas de agrupación

[Caso](#contexto)

[Objetivos](#objetivos)

[Herramientas](#herramientas)

[Conjunto de datos](#datos)

[Actividades a realizar](#actividades)

[Análisis de resultados](#resultados)

[Actividad de verificación](#verificacion)

[Bonos opcionales](#bonos)

[Entregables](#entregables)

[Criterios de evaluación](#rubrica)

[Uso de IAG en actividades del curso ISIS2611](#principios)

## <a name="contexto"></a> Caso: DermAlpes

DermAlpes es una empresa de telemedicina que conecta centros de salud de municipios pequeños con dermatólogos ubicados en las ciudades principales. En los centros de salud, el personal toma fotografías de lesiones de piel con un dermatoscopio y las envía para que un especialista las revise a distancia. Para agilizar esta revisión, DermAlpes quiere incorporar una herramienta que delimite automáticamente la zona de la lesión en cada imagen, de modo que el dermatólogo pueda medir su tamaño y forma, y concentrar su atención en la región relevante.

La empresa no cuenta con imágenes etiquetadas de sus propios pacientes, por lo que, como primera aproximación, quiere evaluar si es posible segmentar la lesión sin entrenar un modelo supervisado, agrupando los píxeles de la imagen según su color y su posición. Para validar esta idea, DermAlpes los ha contratado para construir un método de segmentación sobre un conjunto de imágenes públicas que sí incluye máscaras de referencia elaboradas por dermatólogos, y así medir qué tan bien funciona, dónde falla y por qué.

Es importante aclarar que la herramienta no realizará diagnósticos, su único propósito es delimitar la lesión como apoyo para el especialista. Aun así, la empresa quiere entender las limitaciones del método y los riesgos de usarlo con pacientes cuyas características difieran de las de las imágenes con que fue construido.

## <a name="objetivos"></a> Objetivos

- Aplicar algoritmos de agrupación basados en centroides para segmentar imágenes, representando cada instancia como los colores y posiciones de cada píxel para asignarlo a los diferentes grupos.
- Aplicar un protocolo de ajuste y prueba que permita estimar el desempeño del método, sin ajustarlo a las imágenes con las que se evalúa.
- Analizar de manera crítica el comportamiento del método, identificando sus casos de fallo y las limitaciones de un enfoque basado únicamente en color y posición.
- Reconocer los riesgos de aplicar un método construido con un conjunto de datos a una población con características distintas.

## <a name="herramientas"></a> Herramientas

Durante este laboratorio se trabajará con las siguientes herramientas:

- Librerías de Python para el procesamiento y análisis de datos como:
    - NumPy
    - Scikit-Learn
    - Pillow, scikit-image u OpenCV, para la lectura de imágenes.
    - Matplotlib, Seaborn
- Entorno de desarrollo Visual Studio instalado localmente mediante Anaconda o en la nube mediante Google Colab.

## <a name="datos"></a> Conjunto de datos

Se utilizará una selección de 20 imágenes dermatoscópicas de ISIC 2016. Cada imagen viene acompañada de una máscara de referencia, elaborada por especialistas, que identifica los píxeles correspondientes a la lesión. El objetivo del laboratorio es construir un método que genere automáticamente una máscara estimada a partir de los grupos obtenidos mediante K-medias. Las máscaras de referencia no se utilizan para agrupar los píxeles, sino para comparar y evaluar los resultados del método.

Las imágenes se dividen en dos conjuntos: 15 imágenes de ajuste, utilizadas para explorar y seleccionar la configuración del método mediante la comparación de las máscaras estimadas con las de referencia, y 5 imágenes de prueba, reservadas para evaluar su desempeño sin realizar modificaciones adicionales. Esta separación permite estimar qué tan bien funciona el método con imágenes que no se utilizaron durante su ajuste.

Todos los grupos trabajarán con las mismas imágenes para el ajuste y prueba:

| Imágenes de ajuste | | | Imágenes de prueba |
|:---:|:---:|:---:|:---:|
| ISIC_0000085 | ISIC_0000114 | ISIC_0000150 | ISIC_0000087 |
| ISIC_0000086 | ISIC_0000116 | ISIC_0000153 | ISIC_0000105 |
| ISIC_0000089 | ISIC_0000122 | ISIC_0000156 | ISIC_0000110 |
| ISIC_0000094 | ISIC_0000133 | ISIC_0000157 | ISIC_0000123 |
| ISIC_0000108 | ISIC_0000137 | ISIC_0000112 | ISIC_0000128 |

*Preparación de las máscaras.* Las máscaras originales contienen valores de 0 (piel o fondo) y 255 (lesión), que deben convertirse a 0 y 1, respectivamente. Para comparar las máscaras estimadas con las de referencia, ambas deben tener las mismas dimensiones. Si se reduce el tamaño de las imágenes, las máscaras también deben redimensionarse utilizando interpolación por vecino más cercano, para conservar sus valores binarios.

Fuente: ISIC 2016 ([enlace](https://challenge.isic-archive.com/landing/2016/37/)), tarea de segmentación de lesiones. Citen la fuente del conjunto de datos en el notebook.

**Condición del protocolo:** las imágenes de prueba y sus máscaras no se usan para elegir lod parámetros del método (λ o β), diseñar o ajustar el criterio de construcción de las máscaras, ni para seleccionar K. Solo se consultan al final, para calcular las métricas.

## <a name="actividades"></a> Actividades para realizar.

- **1. Pipeline de preparación y representación.** Construyan un pipeline de Scikit-learn que represente cada imagen mediante una matriz con un píxel por fila y las características [β·R, β·G, β·B, λ·x, λ·y], con los valores de color y las coordenadas normalizados al intervalo [0, 1]. Los parámetros β y λ controlan el peso del color y de la posición, respectivamente, y deben poder modificarse sin cambiar la implementación. Cuando β = 0 y λ ≠ 0, se utiliza únicamente la posición; cuando λ = 0 y β ≠ 0, únicamente el color. Preparen las máscaras de referencia siguiendo las indicaciones de la sección **Conjunto de datos**.

- **2. Agrupación y selección de K.** Para cada imagen de ajuste, exploren diez combinaciones de β y λ con diferentes pesos relativos, incluyendo una que utilice únicamente el color (λ = 0, β ≠ 0) y otra únicamente la posición (β = 0, λ ≠ 0). Para cada combinación, apliquen K-medias con K entre 2 y 8 y seleccionen K mediante el coeficiente de silueta, calculado sobre una muestra reproducible de píxeles. Documenten el tamaño de la muestra y la semilla utilizada.

    *Observación.* K-medias y el coeficiente de silueta dependen de las distancias relativas entre los píxeles. Por eso, multiplicar β y λ por la misma constante no cambia los grupos obtenidos, lo que importa es la razón λ/β. Por ejemplo, (β = 1, λ = 0,5) y (β = 2, λ = 1) producen el mismo resultado. Elijan combinaciones con razones λ/β distintas. 

- **3. Construcción de la máscara de la lesión.** K-medias agrupa los píxeles, pero no identifica cuáles corresponden a la lesión. Diseñen y justifiquen un criterio automático que, a partir de las características de los grupos (por ejemplo, color, tamaño o ubicación), permita identificar los que representan la lesión y construir una máscara binaria. Pueden comparar dos criterios.

    *Observación.* El coeficiente de silueta permite seleccionar K a partir de las características de los píxeles, sin utilizar las máscaras de referencia.

- **4. Ajuste del método.** Utilizando únicamente las 15 imágenes de ajuste, evalúen las combinaciones de β y λ y los criterios de construcción de la máscara propuestos. Para cada configuración, calculen el coeficiente de Dice entre la máscara estimada y la de referencia, y el V-measure entre los grupos obtenidos y las etiquetas de referencia. Utilicen el Dice promedio como criterio principal para seleccionar la configuración final y el V-measure como información complementaria. Revisen también los casos de fallo y justifiquen su elección. Visualicen la imagen original y los grupos obtenidos para las dos configuraciones con mayor Dice promedio, así como para las configuraciones que utilizan únicamente color o posición. La configuración final debe incluir los valores de β y λ y el criterio de construcción de la máscara. 

    El coeficiente de Dice se calcula como:

    $$DICE = \frac{2|X \cap Y|}{|X| + |Y|}$$

    donde X representa los píxeles de la lesión en la máscara de referencia y Y los identificados como lesión en la máscara estimada. Su valor varía entre 0 (sin superposición) y 1 (coincidencia perfecta). Equivale al F1-score a nivel de píxel, considerando la lesión como clase positiva, y puede calcularse con `f1_score` de Scikit-learn sobre las máscaras aplanadas.

    *Observación.* El V-measure (`v_measure_score` en Scikit-learn) mide la correspondencia entre los grupos obtenidos y las etiquetas de referencia, mientras que Dice evalúa la coincidencia entre la máscara estimada de la lesión y la máscara de referencia. Por eso, una agrupación con buena silueta o V-measure no necesariamente produce una buena máscara de la lesión.

- **5. Evaluación y análisis.** Apliquen la configuración seleccionada, sin modificaciones, a las cinco imágenes de prueba. Presenten una tabla con el K seleccionado, la silueta, el Dice, el V-measure, la precisión y el recall para cada imagen, considerando la lesión como clase positiva. Incluyan la imagen original, la máscara estimada y la de referencia. Comparen el Dice promedio de ajuste y prueba, e identifiquen los principales aciertos y errores. No se espera un Dice alto en todas las imágenes.

- **6. Video.** Elaboren un video de máximo tres minutos, dirigido a DermAlpes y a un público no especializado, que explique qué problema aborda la herramienta, cómo se obtiene la máscara de la lesión a partir de los grupos de píxeles, qué muestran los resultados de prueba, en qué casos falla y qué precauciones deben tenerse al usarla.

## <a name="resultados"></a> Análisis de resultados.

Respondan las siguientes preguntas apoyándose en imágenes, tablas y valores concretos obtenidos en el notebook.

1. ¿Qué efecto tuvo incluir la posición de los píxeles? Expliquen en qué imágenes mejoró o perjudicó la segmentación y por qué.

2. ¿Qué relación encontraron entre K, la silueta y el Dice? ¿Por qué estas métricas no necesariamente conducen a las mismas decisiones?

3. ¿Cómo se comparan los resultados de ajuste y prueba? ¿Por qué es importante no modificar el método después de consultar las máscaras de prueba?

4. Identifiquen dos limitaciones de su modelo y expliquen sus posibles causas a partir de las imágenes y de la representación RGB + posición.

5. En el contexto de DermAlpes, ¿qué consecuencias podrían tener los falsos positivos y falsos negativos? ¿Qué sesgos y problemas de generalización deben considerarse antes de utilizar el método con nuevas poblaciones o condiciones de adquisición?

## <a name="verificacion"></a> Actividad de verificación para este laboratorio (obligatoria).

Antes de elegir la configuración final (actividad 4), pidan a una herramienta de IAG que les recomiende valores de β, λ y K, y un criterio para identificar los grupos que corresponden a la lesión y construir la máscara en imágenes dermatoscópicas. Su consulta **no debe anexar imágenes o mencionar el nombre del conjunto (ISIC)**, debe describir las imágenes. Adjunten el historial de la conversación con la IAG donde se muestre el prompt, la respuesta y sucesivas interacciones. Pongan a prueba la recomendación con sus imágenes de ajuste. En la sección "Uso de herramientas de IA generativa" respondan: ¿la recomendación funcionó en sus imágenes? ¿En qué acertó y en qué falló? ¿Por qué la IAG no podía anticipar el comportamiento del método en sus imágenes?

## <a name="bonos"></a> Bonos opcionales

Cada bono otorga hasta 0.25 puntos adicionales sobre la nota del laboratorio (máximo 0.5 en total, sin superar 5.0). Los bonos solo se califican si las actividades obligatorias están completas.

- **Bono A. Visualización del espacio de representación.** Elijan una imagen y apliquen PCA o t-SNE a una muestra de píxeles representados mediante [β·R, β·G, β·B, λ·x, λ·y]. Presenten dos visualizaciones, coloreadas por clúster y por máscara de referencia. Interpreten coincidencias, diferencias y limitaciones de la proyección.

- **Bono B. Investigación y comparación con mean shift.** Investiguen mean shift y aplíquenlo a una imagen de ajuste, usando una resolución y un tamaño de muestra que permitan ejecutarlo. Construyan una máscara binaria y comparen con K-medias mediante Dice, visualización y tiempo de ejecución. Expliquen cómo funciona el ancho de banda y por qué mean shift no requiere fijar K. No es necesario repetir el protocolo completo en todas las imágenes.

## <a name="entregables"></a> Entregables

- Notebook (*.ipynb y *.html) por Bloque Neón con los nombres de los estudiantes. El Notebook debe estar documentado con las justificaciones de las decisiones tomadas en cada paso y las respuestas a las preguntas planteadas en el apartado "Análisis de resultados". Además, deben ser visibles las ejecuciones de cada celda.
- Video explicativo.
- Si desarrollan alguno de los bonos, deben incluirlo en una sección separada al final del notebook, titulada "Bonos".

Esta entrega debe realizarse máximo el **2 de noviembre, 8:00 p.m.** Recuerda registrar en el grupo asignado en Bloque Neón los dos integrantes que presentan este laboratorio, con el fin de habilitar el enlace de entrega.

Si la entrega la hacen después del **2 de noviembre, 8:00 p.m.** y antes del **3 de noviembre, 2:00 a.m.**, su entrega tendrá una penalización del 30%, lo que significa que será calificada sobre 3.5 y no sobre 5.0. Después de esta última fecha toda entrega tendrá una nota de 0.

## <a name="rubrica"></a> Criterios de evaluación

A continuación, se encuentra la rúbrica de calificación que se utiliza para valorar los entregables:

| Actividad | Porcentaje |
|:---|:---:|
| Pipeline de preparación y representación. | 10% |
| Selección de K mediante silueta. | 10% |
| Criterio de construcción de la máscara. | 15% |
| Ajuste, evaluación y tabla de resultados. | 20% |
| Análisis de resultados. | 30% |
| Video. | 5% |
| Uso y verificación de IAG. | 10% |

## <a name="principios"></a> Uso de IAG en actividades del curso ISIS2611

La información que se presenta a continuación también está publicada en Bloque Neón, en la sección del curso **"Uso de la IA generativa"**.

**Principios que rigen el uso de la IA en el curso**

- **Autoría humana.** El estudiante es el responsable final de todo el contenido entregado, incluyendo código, resultados, análisis y conclusiones.

- **Transparencia.** El uso de IAG debe ser claramente declarado, indicando cómo y para qué se utilizó.

- **Pensamiento crítico.** Las respuestas generadas por IAG no deben aceptarse de forma automática; deben ser evaluadas, contrastadas y, cuando sea necesario, corregidas.

- **Aprendizaje y no sustitución.** La IAG debe apoyar el aprendizaje, no reemplazar el proceso de razonamiento, diseño o toma de decisiones por parte del estudiante.

**Reglas prácticas de uso (obligatorias).**

En todas las actividades donde se utilice la IAG, los estudiantes deberán incluir una sección titulada: "Uso de herramientas de IA generativa".

Esta sección deberá contener:

- Declaración del uso. Nombre de la herramienta utilizada y tipo de uso (ayuda conceptual, generación inicial de código, explicación teórica, depuración, redacción, etc.).

- Prompts utilizados. Se deben documentar los prompts principales, de forma textual o resumida. No es necesario incluir interacciones menores, pero sí aquellas que influyeron directamente en el resultado entregado.

- Análisis crítico del resultado. El estudiante deberá responder, al menos, a dos de las siguientes preguntas: ¿Qué partes del contenido generado fueron correctas y útiles? ¿Qué errores, imprecisiones o limitaciones se identificaron? ¿Qué decisiones técnicas fueron modificadas respecto a la respuesta de la IAG y por qué? ¿Qué conceptos del curso permitieron evaluar o mejorar la respuesta generada?

- Aportes propios del estudiante. Debe explicitarse claramente: ¿Qué fue desarrollado, modificado o decidido por el estudiante? ¿Qué ajustes se realizaron sobre el código o la explicación original? ¿Qué aprendizajes se obtuvieron del proceso?

**¿Qué se considera un buen uso y un mal uso?**

**A. Usos recomendados.** Se recomienda el uso de IA generativa para:

- Aclarar conceptos teóricos complejos.
- Comparar enfoques o algoritmos.
- Generar esqueletos iniciales de código (que luego deben ser comprendidos y adaptados).
- Identificar errores de implementación y proponer soluciones.
- Mejorar la redacción técnica de reportes (sin alterar el contenido conceptual).

**B. Usos no recomendados.** No se considera un uso responsable:

- Copiar y entregar código o texto sin comprensión.
- Utilizar IAG para definir decisiones clave sin justificación.
- Presentar resultados sin saber explicar cómo fueron obtenidos.
- Usar IA para responder evaluaciones individuales no autorizadas.
- Omitir la declaración de uso de IA cuando esta fue utilizada.

**Uso de IAG en la elaboración del enunciado de este laboratorio.**

El equipo docente comprometido con el uso transparente de IAG, declara de forma explícita que, en el contexto de este laboratorio, la IAG fue utilizada como apoyo en la búsqueda del conjunto de datos y en la redacción del enunciado. Todo el contenido fue revisado, verificado y ajustado por el equipo docente.
