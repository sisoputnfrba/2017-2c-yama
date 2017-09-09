# Proceso YAMA

Este proceso es el encargado de resolver la solicitud de un Master administrando y balanceando la carga de trabajo[^1] de los diversos procesadores de datos (Worker) conociendo y controlando el estado de todos los Masters en el sistema y de cada una de las operaciones que estos realizaron y deben realizar.

El proceso YAMA **es agnóstico de qué operación realiza el proceso Master sobre los datos** sin embargo **es quién conoce dónde y cuándo ejecutar las operaciones** para obtener el resultado deseado.

Para resolver la solicitud de un Master deberá planificar las siguientes operaciones e indicarle al proceso Master la tarea a realizar:
- **Etapa de Transformación**: Debe planificar la ejecución del script de Transformación sobre todos los bloques del archivo guardando los resultados en un archivo temporal de nombre único en la computadora.
- **Etapa de Reducción Local**: Debe planificar la ejecución del script de Reducción sobre los archivos temporales del paso anterior que estén en el mismo Nodo.
- **Etapa de Reducción Global**: Debe planificar la ejecución del script de Reducción sobre los archivos temporales resultantes del paso anterior entre Nodos.
- **Etapa de Almacenamiento Final**: Deberá indicarle al Master el archivo temporal resultante a almacenar en el FileSystem.

## Tabla de Estados

YAMA mantendrá una estructura interna denominada **tabla de estados** con las solicitudes que le envió al Master y su estado:



| Job   | Master | Nodo    |  Bloque   |  Etapa          | Archivo Temporal | Estado     |
|-------|--------|---------|-----------|-----------------|------------------|------------|
| 1     | 1      | Nodo 1  |  8        | Transformación  |  `/tmp/j1n1b8`     | En proceso |
| 1     | 1      | Nodo 3  |  2        | Transformación  |  `/tmp/j1n3b2`     | En proceso |
| 1     | 1      | Nodo 2  |  9        | Transformación  |  `/tmp/j1n2b9`     | En proceso |
| ...   | ...    | ...  |  ...        | ...  | ...     | ... |
| 25    | 3      | Nodo 5  |  2        | Reducción Local   |  `/tmp/j1n2b2`     | Error |
| W     | X      | Nodo Y  |  Z        | Reducción Global  |  `/tmp/jxnybz`     | Finalizado OK |

Los nombres temporales de los archivos, así como los identificadores de cada Job, deberán ser generados automáticamente por el proceso YAMA evitando que los mismos se repitan.

Al recibir de parte del Master notificaciones de que las operaciones concluyeron actualizará los correspondientes estados. Así mismo, YAMA obtendrá el proceso FileSystem toda la información referente a los Nodos sobre los cuales se encuentran los bloques a utilizar, y la utilizará para poder generar la tabla.

Es necesario aclarar, que las entradas de Jobs no se deberán eliminar una vez finalizada su ejecución (sea correcta o no), a fines de evaluar los resultados del Sistema.

## Etapa de Transformación

Ante la solicitud de un Master de procesar un archivo, le solicitará al Filesystem información del mismo y su composición.

Recibirá del Filesystem la lista de bloques que lo componen, con la correspondiente ubicación de sus dos copias y espacio ocupado en el bloque e iniciará la etapa de Transformación.

Como vimos anteriormente, en esta etapa debe enviarle al Master la indicación de que ejecute **el programa de Transformación en todos los bloques que componen el archivo**.

Al contar con dos copias por cada bloque de datos **optará en función de la cantidad de operaciones que esté ejecutando ese Worker** actualmente considerando todos los Masters en ejecución, intentando minimizar la carga del Nodo.

## Balanceo de Carga

El proceso YAMA deberá efectuar la distribución de las tareas en los Nodos. Dicha distribución responderá al algoritmo de balanceo de carga configurado, siendo sus posibles valores Round **Robin**(RR) o **Weighted Round Robin**(WRR).

A diferencia del algoritmo RR, el algoritmo WRR va a efectuar una valoración sobre cada Nodo en función a la cantidad de tareas que se encuentre realizando y haya realizado; eligiendo el que menos tareas se encuentre realizando y, en caso de empate, el que menos cantidad de tareas haya realizado. Es responsabilidad del grupo conocer los posibles límites que pueda conllevar la particular implementación del mismo realizada por el grupo.

### Ejemplo Archivo `misdatos.csv`
| Bloque   | Copia 0            | Copia 1            | Bytes ocupados |
|----------|--------------------|--------------------|----------------|
| 0        | Nodo 1 - Bloque 8  | Nodo 2 - Bloque 11 | 10180          |
| 1        | Nodo 2 - Bloque 12 | Nodo 3 - Bloque 2  | 10201          |
| 2        | Nodo 1 - Bloque 7  | Nodo 2 - Bloque 9  | 10109          |

Como se ve en este ejemplo, podríamos solicitar tres operaciones de Transformación al Nodo 2 dejando **ociosos** a los Nodos 1 y 3 o bien solicitar una operación al Nodo 1, otra al Nodo 2 y otra al Nodo 3 y así **maximizar el paralelismo**.
Un ejemplo completo de una respuesta de esta etapa se ve en la sección [Ejemplo de una solicitud de Transformación](proceso-master.md#ejemplo-de-una-respuesta-a-una-solicitud-de-transformación-de-yama).

**Cuando las tareas de Transformación vayan concluyendo podrá continuar con la etapa de Reducción Local.**

## Fallo de una tarea de Transformación

Podría pasar que una etapa falle, dado que por ejemplo se pierda la conexión con un Nodo. En ese caso YAMA deberá re-planificar dicha operación en la otra copia.

### Etapa de Reducción Local

Al recibir los mensajes del proceso Master notificando que una tarea de Transformación fue concluida deberá revisar en la tabla de estados ha concluído todas las Transformaciones para un mismo nodo, en ese caso puede iniciar las tareas de Reducción Local.

Es importante registrar esta información en la tabla de estados agregando los campos o columnas que considere pertinente.

El propósito de la Reducción Local es minimizar la transferencia de datos por red entre nodos por lo que la Reducción Local debe ejecutarse aún si hay un solo archivo temporal en ese nodo.

Nuevamente un nombre de archivo temporal para el nodo deberá generarse, intentando que el mismo no se repita.
Puede ver un ejemplo de la solicitud de Reducción Local en [este ejemplo](proceso-master.md#ejemplo-de-una-respuesta-a-una-solicitud-de-reducción-local-de-yama).
Si se generase una falla en la tarea, se deberá registrar la misma y abortar el Job.

### Etapa de Reducción Global

Al recibir la última notificación de una tarea de Reducción Local deberá ejecutar la etapa de Reducción Global involucrando los archivos pre-agregados de todos los nodos. Para la misma, se seleccionará un Nodo que actuará como Encargado de realizar la misma. Dicho Nodo será seleccionado por ser el que menos carga de trabajo poseía al momento de empezar la etapa (Least Loaded Node).

Es importante registrar esta información en la tabla de estados agregando los campos o columnas que considere pertinente.
La Reducción es el último paso, el cual una vez finalizado debe dejar un archivo temporal con el resultado final.
Si se generase una falla en la tarea, se deberá registrar la misma y abortar el Job.
Puede ver un ejemplo de la solicitud de Reducción en [este ejemplo](proceso-master.md#ejemplo-de-una-respuesta-a-una-solicitud-de-transformación-de-yama).

### Archivos de Configuración y Logs

El proceso YAMA deberá poseer un archivo de configuración ubicado en una ubicación conocida donde se deberán especificar, al menos, los siguientes parámetros:

```
FS_IP=
FS_PUERTO=
RETARDO_PLANIFICACION=
ALGORITMO_BALANCEO=
```

Queda a decisión del grupo el agregado de más parámetros al mismo. Es menester aclarar que el retardo de la planificación es un elemento meramente para fines de pruebas, dado que en situaciones normales se lo seteará con el valor de 0ms. Para su implementación, se recomienda que el grupo investigue sobre la función [`usleep()`](http://man7.org/linux/man-pages/man3/usleep.3.html).

Además, YAMA deberá registrar toda su actividad mediante un archivo de log, mostrando por pantalla solamente las actualizaciones de la tabla de estados.

## Recarga de la Configuración

YAMA será capaz de volver a cargar la configuración provista ante un posible cambio del retardo de planificación y el algoritmo de balanceo. Eso se realizará enviando a YAMA la señal **`SIGUSR1`**.

Es menester aclarar que el cambio del algoritmo de balaceo será realizado cuando sea necesaria realizar una nueva planificación o una re-planificación por parte de YAMA. Lo mismo aplica para el retardo.

Será responsabilidad del grupo investigar sobre la forma de enviar señales a procesos mediante la consola bash o el programa `htop`.

[^1]: El objetivo ideal es lograr que todos los Workers tengan la misma carga de trabajo
