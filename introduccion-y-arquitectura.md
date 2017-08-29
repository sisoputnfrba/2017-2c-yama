## Introducción​ ​y​ ​Arquitectura
El objetivo de este TP será poder crear una plataforma que permite **paralelizar la carga de trabajo** en varias máquinas físicas, logrando así reducir los tiempos de procesamiento de datos.

YAMA es una plataforma de **procesamiento distribuido de datos que maximiza el rendimiento ejecutando tareas de manera concurrente en diversos nodos​**. Dichos nodos contienen los bloques que componen los archivos a procesar organizados bajo una estructura de filesystem distribuido.

## Patrón Master/Worker

YAMA utiliza una arquitectura Master/Worker, que nos permitirá distribuir la carga del sistema en varios procesadores (Workers) utilizando un coordinador (Master).

**Existirá una instancia del proceso Master en el sistema para cada tarea (Job) de procesamiento que se requiera​**. Dicho proceso Master se conectará con los Workers quienes serán los encargados de _forkearse (multiplicarse)_ para realizar tareas específicas.

Durante el desarrollo notará que algunos componentes de esta arquitectura resultan una caja negra a la hora de desarrollar la solución, por lo que resulta importante respetar la especificación de manera correcta.

![Arquitectura master/worker](/assets/master-worker.png)


## File System Distribuido
El manejo de los datos que consumirá cada Worker también será responsabilidad del sistema. Para ello utilizaremos una arquitectura de File System Distribuido, cuyo objetivo principal es poder distribuir y replicar la información en varios Data Nodes.

Un factor interesante de este tipo de arquitecturas es la posibilidad de garantizar la Alta Disponibilidad. Esto significa que, si uno de los Data Nodes dejase de estar disponible, sería posible acceder a los datos de todos los archivos del File System, ya que los bloques que este contenía se encuentran replicados en otros Data Nodes del sistema.

![Arquitectura FileSystem](/assets/filesystem.png)

El proceso **FileSystem, que contará con una única instancia en el sistema** será el coordinador principal de las operaciones de lectoescritura y el encargado de gestionar el acceso a los archivos almacenados. Tendrá contacto permanente con todas las instancias de los procesos DataNode y conocerá el estado de todos sus bloques de datos.

Por su parte, los procesos **DataNode** serán los encargados de persistir los bloques de datos del FileSystem en su archivo de datos (data.bin). Una vez almacenados los datos de un archivo en el archivo de datos, **estos serán considerados inmutables**, es decir, que no podrán ser modificados.

## Nodo

Para aprovechar al máximo los recursos, cada proceso Worker correrá en una computadora física distinta. Considerando que **enviar un paquete vía red es sustancialmente más lento que consumirlo de memoria principal o de disco** se procurará que el procesamiento de los datos se realice en la computadora física (Nodo) donde se encuentran los mismos.

![Arquitectura Nodos/DataNodes](/assets/nodo.png)

Definiremos entonces al **Nodo** como **la estructura abstracta que agrupa un DataNode, con su archivo de datos y un Worker**. Cada Nodo representará una computadora física distinta, desde la que se realizarán tanto operaciones de almacenamiento de datos (DataNode) y procesamiento (Worker).

Es importante notar que **los Worker consumen datos directamente del archivo de datos del DataNode**. Dicho procedimiento se realiza con la intención de agilizar el acceso a los datos en cada Nodo. Este trabajo práctico intencionalmente rompe el encapsulamiento del FileSystem con el propósito de maximizar la performance.


## Arquitectura del Sistema
Para poder balancear la carga del sistema, garantizando la sinergia entre el almacenamiento de datos y las tareas de procesamientos, utilizaremos un proceso denominado **YAMA**. Habrá una única instancia de este proceso en el sistema y será el encargado de conectarse con el FileSystem para analizar las solicitudes de los procesos Master, pudiendo así planificarlas.
Teniendo en cuenta estos datos, la arquitectura final del TP será la siguiente:

![Arquitectura general de Y.A.M.A.](/assets/arquitectura.png)

## Deployment y Testing del Trabajo Práctico
Al tratarse de una plataforma distribuida, los procesos involucrados podrán ser ejecutados en diversas computadoras. La cantidad de computadoras involucradas y la distribución de los diversos procesos en estas será definida en cada uno de los tests de la evaluación y **es posible cambiar la misma en el momento de la evaluación**. Es responsabilidad del grupo automatizar el despliegue de los diversos procesos con sus correspondientes archivos de configuración para cada uno de los diversos tests a evaluar.

La disposición del trabajo práctico consistirá en **cargar una serie de archivos en el proceso Filesystem y ejecutar una serie de tareas (Jobs) sobre dichos archivos mediante los procesos Masters**. Dependiendo del resultado de dicha ejecución se evaluará el correcto funcionamiento de la plataforma.


## Ejemplo de ejecución de un Job

1. Se le brindará al grupo un archivo de datos, un programa de Transformación de datos y un programa de Reducción de datos.
2. El grupo iniciará el Filesystem, los correspondientes Nodos (DataNode y Worker) y por último el proceso YAMA.
3. El grupo deberá cargar el archivo de datos en el Filesystem para luego iniciar un proceso Master que ejecute los programas de Transformación y Reducción sobre el archivo de datos.
4. Se revisará el archivo resultante de la ejecución del Job para validar el correcto funcionamiento.

Todo esto estará descrito en el documento de pruebas que se publicará cercano a la fecha de Entrega Final. Archivos y programas de ejemplo se pueden encontrar en el repositorio de la cátedra.

Existe también la posibilidad de brindarle al grupo los archivos de datos ya distribuidos en el FileSystem, de forma tal que tan solo iniciando el mismo, se pueda recuperar el estado del sistema.

Finalmente, recordar la existencia de las [Normas del Trabajo Práctico](https://faq.utnso.com/ntp) donde se especifican todos los lineamientos de cómo se desarrollará la materia durante el cuatrimestre.
