# Laboratorio de aplicación inteligente adaptativa

## Identificación

- Integrante — Vicente Cisternas, Joaquin Muñoz
- Curso y sección — Ingeniería Web Avanzada (OII436-1)
- Repositorio — [Laboratorio3WebAvanzada](https://github.com/VicenteCLeon/Laboratorio3WebAvanzada).

## Preparación

Para reproducir los resultados con las dependencias fijadas, use Python 3.14.7,
la versión con la que se validó el proyecto. Abra una terminal en esta carpeta.

```bash
python -m venv .venv
```

En Windows puede usar `py -m venv .venv`.

Activación en PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Activación en CMD:

```bat
.venv\Scripts\activate.bat
```

Activación en macOS o Linux:

```bash
source .venv/bin/activate
```

Instalación y comprobación:

```bash
python -m pip install -r requirements.txt
python -c "import sys; print(sys.executable)"
python app.py --comprobar
```

La comprobación informa 12 recursos, 4 usuarios y una matriz TF-IDF de 12 filas
por 17 términos. En Windows se puede ejecutar directamente
`.venv\Scripts\python.exe` con los mismos argumentos. `requirements.txt` contiene
las versiones verificadas con `python -m pip freeze`.

## I. Catálogo y perfil inicial

Con Python 3.14.7, `python app.py --comprobar` cargó 12 recursos (`R01` a `R12`),
cuatro usuarios y una matriz TF-IDF de 12 filas por 17 términos. El catálogo tiene
las columnas `id`, `titulo`, `descripcion` y `etiquetas`. El identificador permite
seleccionar el recurso, el título y la descripción lo presentan, y solo las
etiquetas se usan para construir la matriz TF-IDF.

Al cargar el CSV, se quitan los espacios al principio y al final de cada campo
obligatorio. La carga rechaza un catálogo sin filas, columnas obligatorias
faltantes, valores vacíos e identificadores de recurso repetidos. En los perfiles,
`intereses` debe ser texto y se admite una cadena vacía.

`crear_perfil` aplica `transform` con el vectorizador del catálogo y devuelve la
primera fila como un vector NumPy de una dimensión. Así, cada posición del perfil
corresponde al mismo término que la posición equivalente en el catálogo.
[Documentación de scikit-learn](https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.TfidfVectorizer.html#sklearn.feature_extraction.text.TfidfVectorizer.transform)

La comprobación de los perfiles iniciales produjo estos resultados:

| Usuario | Intereses declarados | Términos con peso positivo |
|---|---|---|
| `u1` | `python datos pandas` | `datos`, `pandas`, `python` |
| `u2` | `angular componentes interfaces` | `angular`, `componentes`, `interfaces` |
| `u3` | Cadena vacía | Ninguno |
| `u4` | `zzxyz123` | Ninguno |

`u1` y `u2` tienen vectores e historiales independientes. Los vectores de `u3`
y `u4` tienen 17 ceros. Al pedir recomendaciones para cualquiera de estos dos
usuarios, la lista es vacía y la consola invita a declarar intereses presentes
en el catálogo.

## II. Recomendación basada en contenido

Al iniciar la aplicación, `construir_indice` ajusta el vectorizador una vez con
`etiquetas`. `calcular_puntuaciones` compara el vector actual del perfil con cada
fila del catálogo mediante la similitud del coseno y devuelve las puntuaciones
en el mismo orden de las filas.
[Documentación de scikit-learn](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.pairwise.cosine_similarity.html)

`recomendar` selecciona las puntuaciones positivas, las ordena de mayor a menor
y desempata por `id` ascendente. La consola muestra hasta tres resultados con
cuatro decimales y los términos compartidos. El orden se calcula con las
puntuaciones sin redondear.

### Rankings iniciales observados

Resultados de `python app.py` al pedir recomendaciones para `u1` y `u2` antes de
registrar interacciones.

| Usuario | Puesto | ID | Título | Similitud | Términos compartidos |
|---|---:|---|---|---:|---|
| `u1` | 1 | `R02` | Análisis de datos con Pandas | 1.0000 | datos, pandas, python |
| `u1` | 2 | `R12` | Conectar Python con datos SQL | 0.4965 | datos, python |
| `u1` | 3 | `R03` | Gráficos con Python | 0.4326 | datos, python |
| `u2` | 1 | `R07` | Componentes e interfaces en Angular | 1.0000 | angular, componentes, interfaces |
| `u2` | 2 | `R08` | Formularios e interfaces en Angular | 0.5677 | angular, interfaces |
| `u2` | 3 | `R09` | Consumir una API desde Angular | 0.3087 | angular |

En una comprobación del desempate, se usó el perfil `sql` y se pasaron las filas
`R11`, `R10` y `R01` en ese orden. `R10` quedó antes de `R11` porque ambos tenían
la misma similitud (0.5423 al mostrarla). `R01` obtuvo cero y quedó excluido.

## III. Retroalimentación y adaptación

Al marcar **Me interesa**, `actualizar_perfil` combina el 40 % del vector actual
con el 60 % del vector del recurso elegido. Aplica normalización L2 para que la
mezcla tenga norma unitaria antes de la siguiente interacción. `registrar_interes`
guarda el vector y el evento del usuario activo. El ranking se vuelve a calcular
con los mismos 12 candidatos. El valor de `alpha` (0,6) es didáctico.
[Documentación de scikit-learn](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.normalize.html)

### Comparación de la adaptación

En la misma sesión, `u1` consultó sus recomendaciones y marcó **Me interesa** en
`R04` («API web con Python») desde el catálogo completo. Su historial pasó de `[]`
a `[{'numero': 1, 'accion': 'Me interesa', 'recurso': 'R04'}]`. Estos fueron los
pesos de los términos con valor positivo en alguno de los dos perfiles.

| Término | Antes | Después |
|---|---:|---:|
| `api` | 0.000000 | 0.446269 |
| `datos` | 0.465091 | 0.232455 |
| `pandas` | 0.753247 | 0.376477 |
| `python` | 0.465091 | 0.637116 |
| `web` | 0.000000 | 0.446269 |

El ranking inicial `R02`, `R12`, `R03` pasó a `R04` (0.8752), `R02` (0.6880) y
`R09` (0.4958). **`R05` subió de 0.0000 a 0.4378** porque comparte `api` y `web`,
dos términos que entraron al perfil al valorar `R04`. El historial contiene solo
el evento de `R04`, por lo que el cambio en `R05` muestra el efecto del perfil
actualizado sobre otro candidato.

El [CSV de puntuaciones](evidencias/puntuaciones_antes_despues.csv) conserva los
12 IDs y las puntuaciones anteriores, posteriores y sus diferencias. El
[reporte de la demostración](evidencias/reporte_demo.json) registra el historial,
los pesos del perfil y los rankings observados. En una prueba adicional, `R99`
produjo el aviso «No existe el recurso R99.» y el perfil y el historial de `u1`
conservaron sus valores.

## Funciones del motor

| Función | Entrada principal | Salida y requisito |
|---|---|---|
| `crear_perfil` | Texto de intereses y vectorizador ya ajustado | Vector NumPy de una dimensión obtenido con `transform`; no ajustar nuevamente el vocabulario. |
| `calcular_puntuaciones` | Vector del perfil y matriz de recursos | Un valor de similitud del coseno por recurso. |
| `recomendar` | Perfil y catálogo indexado | Hasta 3 resultados con puntuación positiva; orden descendente y desempate por `id` ascendente. |
| `actualizar_perfil` | Perfil anterior y vector del recurso valorado | Vector normalizado de la combinación `0.4 * perfil + 0.6 * recurso`. |

## IV. Validación y entrega

Modo interactivo:

```bash
python app.py
```

Seleccione un usuario. Puede consultar el catálogo, solicitar recomendaciones y
registrar **Me interesa** indicando el ID de cualquier recurso, aunque no aparezca
entre los tres primeros. Un ID desconocido muestra un aviso y no actualiza el perfil.
Los perfiles y sus historiales de interacciones se conservan por separado durante
la ejecución. La consola muestra el historial del usuario activo. Al reiniciar se
cargan nuevamente los intereses iniciales del JSON y los historiales quedan vacíos.

Demostración reproducible:

```bash
python app.py --demo
```

Este comando parte de los perfiles iniciales, ejecuta T1-T4 y genera:

- `evidencias/reporte_demo.json`: perfiles, historiales, recomendaciones y comprobaciones.
- `evidencias/puntuaciones_antes_despues.csv`: los mismos 12 recursos con sus
  puntuaciones anteriores, posteriores y diferencias, incluyendo ceros.

### Resultados de las pruebas

| Caso | Entrada o acción | Resultado esperado | Resultado observado | Estado | Interpretación |
|---|---|---|---|---|---|
| T1 | Consultar u1 y u2 al iniciar. | Hasta tres resultados positivos en orden descendente y desempate por ID ascendente. | u1 obtuvo R02, R12, R03 y u2 obtuvo R07, R08, R09. Los seis puntajes fueron positivos y descendentes en su lista. En la comprobación de empate, R10 quedó antes de R11. | Cumple | Los perfiles priorizan etiquetas distintas sobre el mismo catálogo. El empate se resuelve por ID incluso al invertir el orden de entrada. |
| T2 | Marcar Me interesa en R04 con u1 y recalcular en la misma sesión. | Cambia el vector de u1 y la puntuación de otro recurso entre los mismos 12 candidatos. | Se registró un evento para R04 y el ranking pasó a R04, R02, R09. La similitud de R05 pasó de 0.0000 a 0.4378 en la comparación de 12 filas. | Cumple | R05 ganó relevancia porque el perfil incorporó api y web al valorar R04. |
| T3 | Volver a consultar u2 después de la interacción de u1. | Se conservan el vector, el historial y los puntajes de u2. | El historial de u2 siguió vacío. Sus pesos (componentes 0.657512, interfaces 0.564679, angular 0.498813), los puntajes de los 12 recursos y su ranking coincidieron antes y después. | Cumple | La interacción se aplicó solo al perfil activo. |
| T4 | Recomendar para u3 con intereses vacíos y u4 con zzxyz123. | Ambos reciben lista vacía y un mensaje informativo. | Ambos devolvieron [] y los 12 puntajes fueron cero. La consola mostró «No hay coincidencias positivas. Declare intereses presentes en el catálogo.» para cada uno. | Cumple | Sin términos reconocidos, el perfil tiene peso cero en todas las posiciones y el filtro excluye todos los candidatos. |

## Preguntas evaluadas

1. ¿Qué información describe el contenido de los recursos y qué información constituye el perfil del usuario? Identifique ambas en los archivos del proyecto.

   En datos/recursos.csv, etiquetas describe el contenido para el cálculo de
   similitud. titulo y descripcion presentan cada recurso e id lo identifica.
   En datos/perfiles.json, intereses contiene el texto inicial de cada usuario.
   Durante la ejecución, app.py conserva ese texto y mantiene por usuario un
   vector actual de intereses y un historial de interacciones, inicialmente vacío.

2. ¿Por qué se utiliza fit_transform sobre el catálogo y transform sobre los intereses del usuario? ¿Qué problema causaría ajustar un vectorizador diferente para cada perfil?

   fit_transform aprende el vocabulario y los pesos IDF de las etiquetas del
   catálogo y construye su matriz. transform representa cada perfil con ese
   mismo vocabulario y esos mismos pesos. Un ajuste por perfil podría cambiar
   la cantidad de columnas, el término asociado a cada posición y los pesos
   IDF. El coseno requiere que ambos vectores compartan esas características
   para que la comparación tenga sentido.

3. ¿Por qué el mismo catálogo puede producir recomendaciones distintas para u1 y u2? ¿Una similitud de 0,80 significa que existe un 80 % de probabilidad de que el recurso guste? Justifique.

   Los intereses de u1 dan peso a python, datos y pandas, mientras que los
   de u2 dan peso a angular, componentes e interfaces. Por eso, la
   dirección del vector de R02 coincide con la de u1 y la de R07 con la de
   u2. Un valor de 0,80 mide la cercanía entre las direcciones de dos vectores
   TF-IDF. Para estimar la probabilidad de que el recurso guste, harían falta
   valoraciones observadas y un modelo probabilístico calibrado con ellas.

4. ¿En qué se diferencia modificar manualmente los intereses iniciales de la adaptación implementada? Identifique el evento que provoca el cambio, el estado modificado y su efecto observable.

   Editar intereses en datos/perfiles.json cambia el texto con el que se
   construye el perfil al iniciar la aplicación. En la adaptación, el evento
   Me interesa cambia automáticamente el vector actual del usuario activo
   en memoria. La aplicación registra el evento y recalcula los puntajes y las
   recomendaciones en la misma sesión.

5. Relacione su implementación con las etapas del ciclo MAPE-K estudiado en clase: Monitorizar, Analizar, Planificar, Ejecutar y Conocimiento. Indique una función, operación o dato de su programa para cada etapa.

   | Etapa | Operación o dato del programa |
   |---|---|
   | Monitorizar | interactivo captura la acción Me interesa y el ID indicado para el usuario activo. |
   | Analizar | registrar_interes comprueba el usuario y usa obtener_indice para validar el ID y localizar el vector del recurso en el catálogo. |
   | Planificar | actualizar_perfil aplica la regla fija que asigna un peso de 0,4 al perfil actual y de 0,6 al recurso valorado. |
   | Ejecutar | Se normaliza la mezcla, se guarda el nuevo vector con el evento en el historial y recomendar recalcula el ranking. |
   | Conocimiento | El catálogo, el vectorizador y la matriz TF-IDF representan los recursos. El vector actual y el historial conservan el estado de cada usuario. |

6. Indique una limitación del mecanismo implementado y proponga una mejora concreta. Explique qué dato adicional o qué cambio del algoritmo exigiría esa mejora.

   El perfil actualizado se pierde al cerrar la aplicación. Para conservarlo,
   guardaría en un archivo JSON las valoraciones de cada usuario y volvería a
   aplicarlas en el mismo orden al iniciar.

### Reproducción desde una copia nueva

bash
git clone https://github.com/VicenteCLeon/Laboratorio3WebAvanzada.git
cd Laboratorio3WebAvanzada


Cree y active el entorno de la sección Preparación. Instale requirements.txt,
ejecute python app.py --comprobar y luego python app.py --demo. Este último
comando regenera el CSV y el JSON en evidencias/.
