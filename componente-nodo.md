# Componente Nodo

El nodo en este trabajo práctico es un concepto abstracto compuesto por los procesos Worker y DataNode ejecutando en la misma computadora. Ambos comparten archivo de configuración y solo un Nodo puede ser ejecutado por computadora.
- DataNode
- Worker
- Archivo de Datos (data.bin)

![Un componente Nodo](/assets/componente-nodo.png)

Como muestra el diagrama el archivo de datos es accedido por ambos procesos de manera concurrente. No hay inconvenientes de sincronización dado que el acceso del Worker es únicamente para lectura de bloques previamente escritos.

Ambos procesos son agnósticos del contenido de data.bin, únicamente cumplen con el objetivo de almacenar/leer bloques (en el caso del DataNode) y procesar bloques en el caso del Worker.

### Ejemplo de archivo de configuración del Nodo
```
IP_FILESYSTEM=192.168.1.254
PUERTO_FILESYSTEM=5040
NOMBRE_NODO=NODO1
PUERTO_WORKER=5050
RUTA_DATABIN=/home/utnso/data.bin
```

## DataNode
Al iniciar este proceso, leerá el archivo de configuración del nodo, abrirá el archivo data.bin y deberá conectarse al proceso Filesystem, identificarse y quedar a la espera de solicitudes por parte del proceso Filesystem respetando la interfaz descrita a continuación:

### Interfaz con el FileSystem

El FileSystem podrá solicitar las siguientes operaciones:
- `getBloque(numero)`: Devolverá el contenido del bloque solicitado almacenado en el Espacio de Datos.
- `setBloque(numero, [datos])`: Grabará los datos enviados en el bloque solicitado del Espacio de Datos

Para simplificar el acceso a los datos de forma aleatoria el alumno debe investigar e implementar la llamada al sistema mmap().

### Archivo de Log

El proceso DataNode deberá contar con un archivo de configuración en el cual se logearán todas las operaciones realizadas. Las mismas deberán ser mostradas por pantalla.

## Worker

El Worker es un proceso encargado de aplicar operaciones sobre bloques de archivos o sobre archivos temporales. Las operaciones son recibidas por parte del Master quien se conecta al Worker para ejecutar una tarea comandada por YAMA.

Al iniciar leerá el archivo de configuración del nodo y quedará a la espera de conexiones por parte de procesos Master.
Al recibir dicha conexión Worker creará un proceso hijo ( [`fork()`](http://man7.org/linux/man-pages/man2/fork.2.html) ) para atender la solicitud el cual terminará al concluir la operación. Es menester aclarar que no se deberán crear hilos para atender las conexiones.

Recibirá del proceso Master, independientemente de en qué etapa se encuentre, un código a ejecutar, un origen de datos y un destino.

En el caso de una etapa de Transformación el origen debe ser una porción del archivo data.bin (un bloque de datos) y el destino un archivo temporal, en el caso de una operación de Reducción el origen y el destino son archivos temporales del filesystem local del Nodo.

El archivo de procesamiento es un programa o un script que debe ejecutarse localmente en el nodo, recibiendo por entrada estándar (`STDIN`) el origen y almacenando el resultado expuesto por salida estándar (`STDOUT`) en el correspondiente destino[^7].

![Esquema de proceso Worker](/assets/worker-pipe.png)

El Worker es completamente agnóstico de lo que cada programa realiza sobre los datos. Su función es pasarle por entrada estándar el contenido indicado por el Master y almacenar la salida estándar en el archivo temporal indicado por el Master.

Existen particularidades del procesamiento dependiendo en qué etapa nos encontremos:

### Etapa de Transformación

En esta etapa existe una particularidad en la ejecución de la rutina de Transformación donde, para poder lograr un eficiente procesamiento distribuído, **el resultado debe ser almacenado de manera ordenada (`sort`)**.

![Worker con sort](/assets/pipeline.png)

### Etapa de Reducción Local

En esta etapa se asume que los datos recibidos están ordenados de manera alfabética ya que el paso anterior lo hizo antes de almacenarlos.
Para esta etapa los diversos archivos temporales deben ser apareados antes de ser enviados al programa de Reducción.

![Reduccion Local](/assets/reduccion-local.png)

### Etapa de Reducción Global

Para esta etapa se asume que los archivos nuevamente están ordenados alfabéticamente por lo que en este caso el apareo debe ocurrir con los archivos entre nodos antes de ser enviados al programa de Reducción

![Reduccion Global](/assets/reduccion-global.png)

Para esta etapa, los Nodos quedarán a la espera de una conexión de Master (si fuesen elegidos Encargados por YAMA) o una conexión de otro Nodo.

### Archivo de Log

El proceso DataNode deberá contar con un archivo de configuración en el cual se logearán todas las operaciones realizadas. Las mismas deberán ser mostradas por pantalla.
Archivo de Datos (data.bin)

El Archivo de Datos del Nodo será un archivo almacenado localmente de tamaño pre-solicitado. El contenido del mismo será exclusivamente los bloques de datos y no podrá contener estructuras administrativas.

Es decir para que un Nodo disponga de un espacio de datos de 20MB, antes de iniciar los procesos del Nodo por primera vez, se deberá generar un archivo local de 20MB llamado data.bin. Luego el primer MB de este archivo corresponden al bloque 0 del nodo, el segundo MB al bloque 1 y sucesivamente.
Tanto el proceso Worker como el DataNode deben obtener el tamaño total del archivo de sus propiedades (ver syscall [`stat()`](https://en.wikipedia.org/wiki/Stat_(system_call)) ).

En general para acceder al bloque `X`, se acceder a este archivo en la posición `X * 1 MB`.

![Bloques](/assets/data-bin.png)

[^7]: Lectura recomendada: [https://github.com/nicozar95/ejemplo_correr-script_so](https://github.com/nicozar95/ejemplo_correr-script_so)
