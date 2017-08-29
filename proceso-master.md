# Proceso Master

Un proceso Master permitirá al usuario **ejecutar una tarea (Job)** sobre un sólo archivo de datos almacenado en el Filesystem. Dicha tarea nos permitirá procesar los datos de un archivo del FileSystem mediante la aplicación de dos operaciones: Transformador y Reductor.

Estas operaciones resultan de caja negra para el alumno y serán provistas por la cátedra como archivos externos (binario o script)

**El resultado de la aplicación de estos programas será guardado en formato de archivo en el FileSystem**. Para ello, se deberá indicar la ruta de almacenamiento del archivo resultante.

Ejemplo:
`./Master /scripts/transformador.sh /scripts/reductor.rb yamafs:/datos.csv yamafs:/analisis/resultado.json`

_Se observa que el comando inicia un proceso Master para ejecutar el script transformador.sh y el script reductor.rb sobre el archivo datos.csv en yamafs. El resultado deberá ser almacenado en /analisis/resultado.json también en yamafs._

Cabe destacar **que utilizaremos el prefijo “yamafs:” para referirnos a los archivos de nuestro FileSystem**, evitando de esta forma las ambigüedades posibles entre los archivos locales y los que se encuentran cargados en nuestro sistema. Por ejemplo: `yamafs:/datos.csv`.

A través de la plataforma YAMA, el programa Transformador analizará los datos de los **bloques** (ver [FileSystem](proceso-filesystem.md)) de un archivo y generará como resultado un archivo temporal. Luego el programa Reductor, tomará esos archivos temporales y los **unificará** en un único archivo final.

La característica principal de estos programas es que múltiples instancias de estos se ejecutarán de forma paralela en todo el sistema. Para ello, **el proceso Master se encargará de distribuir a los distintos Workers el programa el Transformador y el programa Reductor e indicarles a cada uno qué operación deben hacer sobre qué bloque u archivo**.

## Etapas del proceso Master
Al iniciar, leerá su archivo de configuración, se conectará al Proceso YAMA, le indicará el archivo sobre el que desea operar y quedará a la espera de indicaciones de YAMA.
Para resolver dicha solicitud YAMA ejecutará tres etapas una a continuación de la otra:

- Etapa de Transformación
- Etapa de Reducción Local
- Etapa de Reducción Global

_Debido a que la responsabilidad de la ejecución de las etapas está dividida entre el proceso YAMA y el proceso Master es conveniente, para una mejor comprensión del enunciado, leer la responsabilidad de cada proceso en cada etapa de manera conjunta._

## Etapa de Transformación

En esta etapa el proceso Master se conectará a los Workers en los nodos correspondientes y le indicará cuáles son los bloques sobre los que se deberá aplicar el programa de Transformación y dónde guardar sus resultados:

En esta etapa el proceso YAMA le indicará:
- A qué procesos Worker deberá conectarse con su IP y Puerto
- Sobre qué bloque de cada Worker debe aplicar el programa de Transformación.
- El nombre de archivo temporal donde deberá almacenar el resultado del script de Transformación.

El proceso Master deberá entonces:
1. Iniciar un hilo por cada etapa de Transformación indicada por el proceso YAMA.
2. Cada hilo se conectará al correspondiente Worker, le enviará el programa de Transformación y le indicará el bloque sobre el cuál quiere ejecutar el programa, la cantidad de bytes ocupados en dicho bloque y el nombre del archivo temporal donde guardará el resultado.
Quedará a la espera de la confirmación por parte del proceso Worker.
3. Notificara del éxito o fracaso de la operación al proceso YAMA.

### Ejemplo de una respuesta a una solicitud de Transformación de YAMA

| Nodo               | IP y Puerto del Worker  | Bloque  |  Bytes Ocupados   |  Archivo Temporal |
|--------------------|-------------------------|---------|-------------------|-------------------|
| Nodo 1    | 192.168.1.10:5000     | 38   | 10180   |  `/tmp/Master1-temp38` |
| Nodo 1  |  192.168.1.10:5000    |  39 | 1048576  |  `/tmp/Master1-temp39` |   |
| Nodo 2  |   192.168.1.11:5555   |  44  | 1048576   |  `/tmp/Master1-temp44` |
| Nodo 2  |   192.168.1.11:5555   |  39  |  1048576  | `/tmp/Master1-temp39`  |
| Nodo 2 |    192.168.1.11:5555  |  46  | 1048576   |  `/tmp/Master1-temp46`  |


_Note que en el ejemplo se ejecutarán dos transformaciones en el Worker del Nodo 1 y tres en el Worker del Nodo 2 por ende habrá 5 archivos temporales en 2 computadoras_

**Cuando en un Nodo se ejecutaron todas las operaciones de Transformación, inmediatamente pasará a la Etapa de Reducción Local.**

## Etapa de Reducción Local

En esta etapa el proceso Master deberá indicarle a los Workers correspondientes **cuáles son los archivos temporales sobre los cuales deberá aplicarse el programa de Reducción**, y dónde guardar sus resultados.

En esta etapa el proceso YAMA le indicará:
- Nombre de los archivos temporales almacenados en cada uno de los Nodos que deberá procesar con el programa de Reducción
- Nombre de archivo temporal de reducción local por cada Nodo.


### Ejemplo de una respuesta a una solicitud de reducción local de YAMA

| Nodo   | IP y Puerto del Worker | Archivo Temporales de Transformación | Archivo Temporal de Reducción Local |
|--------|------------------------|--------------------------------------|-------------------------------------|
| Nodo 1 | 192.168.1.10:5000      | `/tmp/Master1-temp38`                | `/tmp/Master1-Worker1`              |
| Nodo 1 | 192.168.1.10:5000      | `/tmp/Master1-temp39`                | `/tmp/Master1-Worker1`              |
| Nodo 2 | 192.168.1.11:5555      | `/tmp/Master1-temp44`                | `/tmp/Master1-Worker2`              |
| Nodo 2 | 192.168.1.11:5555      | `/tmp/Master1-temp39`                | `/tmp/Master1-Worker2`              |
| Nodo 2 | 192.168.1.11:5555      | `/tmp/Master1-temp46`                | `/tmp/Master1-Worker2`              |

_Note que en el ejemplo se ejecutarán dos reducciones locales. Una en el Worker del Nodo 1 y otra en el Worker del Nodo 2 con los archivos temporales correspondientes al paso anterior._

El proceso Master deberá:
1. Iniciar un hilo por cada Nodo y  conectarse al Worker.
2. Enviar el programa de Reducción, la lista de archivos temporales del Nodo y el nombre del archivo temporal resultante.
3. Quedará a la espera de la confirmación por parte del proceso Worker.
4. Notificara del éxito o fracaso de cada etapa al proceso YAMA.

De esta forma, el proceso Master **esperará a que todos los Workers terminen su etapa de Reducción Local**. Cuando esto suceda, **comenzará con la Etapa de Reducción Global**.

## Etapa de Reducción Global

En esta etapa el proceso Master deberá indicarle al Worker encargado por YAMA que debe realizar la Reducción Global sobre todos los archivos temporales de la etapa de Reducción Local, y dónde guardar almacenar el resultado final.

Resulta importante destacar que, si bien un solo Worker será el encargado de realizar la Reducción Global, este deberá conectarse con sus homónimos para realizar dicha operación. Se detalla este procedimiento en el apartado correspondiente al Worker.

El proceso YAMA le indicará:
- Lista de Nodos con la IP y Puerto del Worker.
- El nombre de archivo temporal de Reducción de cada Worker.
- Un Worker designado como encargado.
- La ruta donde guardar el archivo resultante de la Reducción Global.

### Ejemplo de una respuesta a una solicitud de Reducción Global de YAMA

| Nodo   | IP y Puerto del Worker | Archivo Temporal de Reducción Local |  Archivo de Reducción Global | Encargado |
|--------|------------------------|--------------------------------------|------------------------------|-----------|
| Nodo 1 | 192.168.1.10:5000      | `/tmp/Master1-Worker1` | | |
| Nodo 2 | 192.168.1.11:5555      | `/tmp/Master1-Worker2` | `/tmp/Master1-final` | SI |

_Note que en el ejemplo vemos los archivos temporales de Reducción Local del ejemplo del paso anterior. De manera arbitraria YAMA designó al Worker2 como encargado._

El proceso Master deberá:
1. Conectarse al Worker encargado.
2. Enviar la rutina de Reducción, la lista de procesos Worker con sus IPs y Puertos y los nombres de los archivos temporales de Reducción Local.
3. Quedará a la espera de la confirmación por parte del proceso Worker.
4. Notificara del éxito o fracaso de la operación al proceso YAMA.

Al finalizar la ejecución de esta Etapa, el proceso Master le comunicará al proceso YAMA y este le indicará que realice el **Almacenado Final**.

## Almacenado final

Para finalizar la tarea se almacenará el archivo resultante del Job en el Filesystem distribuído para ser accedido por el usuario o por otro Job con el nombre y la ruta especificadas por el usuario al ejecutar el proceso Master.

El proceso YAMA le indicará:
- La IP y Puerto del Worker al cual conectarse
- El nombre del archivo resultado de la Reducción Global.


### Ejemplo de una respuesta a una solicitud de almacenado final de YAMA
| Nodo               | IP y Puerto del Worker  |  Archivo de Reducción Global |
|--------------------|-------------------------|------------------------------|
| Nodo 2  |   192.168.1.11:5555   |  `/tmp/Master1-final` |

El proceso Master entonces le solicitará al Worker encargado que se conecte al Filesystem y le envié el contenido del archivo de Reducción Global y el nombre y ruta que este deberá tener.

*Concluida esta operación se da por terminado el Job y el proceso Master terminará.*
Si la operación de escritura del archivo en el FileSystem fallara, el Worker deberá informar el error al proceso Master, quien a su vez deberá notificarlo a YAMA.


## Replanificación

Puede suceder que un proceso Worker finalice su ejecución de forma abrupta o con errores. Si este fuera el caso, deberá notificarse a YAMA y este será encargado de re-planificar sólamente las tareas de transformación que hayan fallado en un nodo que contenga una copia de los datos.

Cabe aclarar que YAMA puede definir que **no existe posibilidad de replanificar las etapas que fallaron, y dar por finalizado el Job**. Como por ejemplo, en casos donde no se posean nodos con copias de los datos, o se encuentre en una etapa de Reducción o Almacenamiento Final.

## Sobre el proceso Master

Es esperable que varias instancias del proceso Master se encuentren ejecutando de manera simultánea en el sistema.
Tanto si el proceso Master llegase a tener un error de ejecución o la ejecución concluyera de manera satisfactoria **no se requiere que se eliminen los archivos temporales producidos por las diversas etapas**.


## Métricas

El proceso Master mostrará luego de concluir su ejecuciòn, las siguientes métricas:
1. Tiempo total de Ejecución del Job.
2. Tiempo promedio de ejecución de cada etapa principal del Job (Transformaciòn, Reducción y Reducción Local).
Can3. tidad máxima de tareas de Transformación y Reducción Local ejecutadas de forma paralela.
4. Cantidad total de tareas realizadas en cada etapa principal del Job.
5. Cantidad de fallos obtenidos en la realización de un Job.

## Archivo de Configuración y Logs

El proceso Master deberá poseer un archivo de configuración ubicado en una ubicación conocida donde se deberán especificar, al menos, los siguientes parámetros:

```
YAMA_IP=
YAMA_PUERTO=
```

Queda a decisión del grupo el agregado de más parámetros al mismo.
Además, el proceso Master deberá registrar toda su actividad mediante un archivo de log, mostrando los mismos por pantalla y finalizando mostrando sus métricas.
