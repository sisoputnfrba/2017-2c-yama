# Anexo II - Algoritmos de Planificación

## Historia Previa

A raíz de varias consultas sobre el funcionamiento del funcionamiento del algoritmo Round Robin y Weighted Round Robin; la cátedra decidió realizar una estandarización de una implementación de los mismos bajo un nombre diferente, a fin de evitar confusiones con los vistos en la teoría de Planificación de Procesos: Clock y W-Clock. Para ello, primero hemos de realizar una pequeña introducción sobre las particularidades de estos algoritmos aplicados al Trabajo Práctico.

### (Pre?) Planificación
Existen diferencias entre el algoritmo que se ve en la teoría y la implementación necesaria para este Trabajo Práctico. La principal diferencia radica en que lo denominado *planificación* es, en realidad, una **pre-planificación**.

En un sistema convencional nuestro sistema puede *quitar un recurso a un proceso y asignárselo a otro*, o bien esperar a que un *proceso desaloje* un recurso para planificarlo. Esto significa que, durante las etapas de planificación, nuestro sistema **reasigna recursos**. 

Dicho concepto, **no existe en el trabajo práctico**, ya que no sólo, los mismos Nodos son capaces de procesar múltiples tareas de Jobs al simultáneo, sino que además, las tareas de reducción deben ejecutarse en los mismos Nodos que las tareas de transformación. Esto contamina el concepto anterior y evitando que la ejecución pueda ser cortada y resumida en otro Nodo.

## Detalle de los Algoritmos
Ambos algoritmos poseerán un modo de ejecución similar, diferenciándose en la forma de obtención de la función de Availability (o disponibilidad) de los Workers, que determinará cuántas operación-bloque a realizar.

### Preconceptos
En esta pre-planificación, los recursos sobre los cuales trabajar en por YAMA serán los Workers, siendo la unidad de pre-planificación la tarea a efectuarse sobre el bloque de un archivo en particular.

El algoritmo se compone de tres partes:
- Una **función de Availability** o Disponibilidad
- Un puntero a la instrucción base, denominado **Clock**, de funcionamiento similar al del algoritmo de Memoria visto en la teoría que lleva el mismo nombre.
- Un conjunto de **Reglas** a seguir.

### Obtención de la función de Availability
Como se dijo anteriormente, dado que el funcionamiento del Clock y el set de Reglas son idénticos tanto para el algoritmo Clock como para W-Clock, *la diferenciación radica en su función de disponibilidad*.
Para poder definir la misma, **es necesario conocer el valor del conocido Disponibilidad Base**, que deberá estar detallado en el archivo de configuración de YAMA, y podrá cambiar en tiempo de ejecución, siendo tomado en cuenta para la próxima pre-planificación.

La función de Availability de un Worker, definida como A(w) responderá a lo siguiente:

**A(w) = Base + PWL(w)**

*Siendo PWL(w)=0* bajo el algoritmo *Clock* y, bajo el algoritmo *W-Clock*, será la disponibilidad del Worker en función de su carga de trabajo actual. En este segundo caso, responderá a la fórmula:

**PWL(w) = WLmax - WL(w)**

*Siendo*
*WLmax*: la mayor carga de trabajo existente entre todos Workers
*WL(w)*: la carga de trabajo actual del Worker. 

**Nota**: La cargas de trabajo deberán corresponder a un *número entero sin signar de 32bits*.

### Reglas de Funcionamiento

1. Inicio:
   - Se calcularán los valores de disponibilidades para cada Worker
   - Se posicionará el Clock en el Worker de mayor disponibilidad, *desempatando por el primer worker que tenga menor cantidad de tareas realizadas históricamente*.
2. Se deberá evaluar si el Bloque a asignar se encuentra en el Worker apuntado por el Clock y el mismo tenga disponibilidad mayor a 0.
   - Si se encuentra, se deberá reducir en 1 el valor de disponibilidad y avanzar el Clock al siguiente Worker. Si dicho clock fuera a ser asignado a un Worker cuyo nivel de disponibilidad fuera 0, *se deberá restaurar la disponibilidad al valor de la disponibilidad base y luego, avanzar el clock al siguiente Worker, repitiendo el paso 2*.
   - En el caso de que no se encuentre, se deberá utilizar el siguiente Worker que posea una disponibilidad mayor a 0. *Para este caso, no se deberá modificar el Worker apuntado por el Clock*.
   - Si se volviera a caer en el Worker apuntado por el clock por no existir disponibilidad en los Workers (es decir, da una vuelta completa a los Workers), *se deberá sumar al valor de disponibilidad de todos los workers el valor de la __Disponibilidad Base__ configurado al inicio de la pre-planificación*.
3. Si existe otro Bloque sobre el cual realizar una tarea, efectuar el paso (2) nuevamente con el nuevo bloque. 

Una vez finalizado el paso (3), se habrá obtenido la planificación a entregar al Master. Es necesario recalcar que la creación de los planes de ejecución se realizará de forma secuencial, obteniendo cada Job su propia planificación de tareas.

### Actualización de la Carga de los Workers

Tras haber finalizado la creación del plan de ejecución, es necesario que YAMA actualice la carga de trabajo Worker (WL(w)), importantes para la ejecución del algoritmo W-Clock, siendo una unidad por cada Bloque sobre el cual ejecutar una transformación + reducción local.

Como caso particular durante la Reducción Global, a la hora de elegir un Worker Encargado se utilizará el que menor WL(w) tuviera al efectuar dicha etapa. Además, se deberá sumar una cantidad igual a la mitad (redondeando hacia arriba) de la cantidad de archivos temporales a reducir, a la carga de trabajo Worker Encargado, a modo de denotar el diferente peso de las tareas a realizar entre los Workers en dicha etapa.

Finalmente, luego de que se finalice un Job, se deberá reducir la WL(w) la cantidad de unidades que se sumaron, a fin de reducir la carga de trabajo del Worker.

## Ejemplos

Supongamos un Archivo compuesto por 6 bloques, con la siguiente distribución de los Bloques:
	DataNode1: 0 1 3 4 6
	DataNode2: 0 2 3 5 6
	DataNode3: 1 2 4 5
y un valor de Disponibilidad Base de 2, veamos cómo se comportan los algoritmos. Bajo la primera ejecución, tanto el Clock como el W-Clock se comportarán de forma idéntica. Veamos cómo resultaría:

![Ejemplo 1](/assets/clock-ejemplo-1.png)

Como vemos, terminaría generando una carga de trabajo en los Workers de 3 para el primero y 2 para los restantes. Si lanzamos un nuevo Job que actuara sobre el mismo archivo, mediante el algoritmo Clock obtendremos el mismo resultado, generando una carga de trabajo para los Workers de 6 y 4 para los restantes y así sucesivamente.

Sin embargo, si utilizamos el algoritmo W-Clock que toma en cuenta la carga de trabajo, obtendremos lo siguiente:

![Ejemplo 2](/assets/clock-ejemplo-2.png)

Primero que nada, al cambiar los valores de Disponibilidad [A(w)], el Clock arrancaría apuntando al primer Worker de mayor carga de trabajo. Además, el funcionamiento estará atado al los diferentes valores de disponibilidad, a diferencia del algoritmo Clock.

El resultado de la ejecución del W-Clock resultaría en una carga de trabajo final de 5 para los 2 primeros Workers y 4 para el tercero.

Algo a notar, es que en contraste con el algoritmo Clock, si ejecutamos un tercer Job sobre, por ejemplo, los mismos archivos mientras se estuvieran ejecutando los otros 2, obtendrìamos algo como lo siguiente:

![Ejemplo 3](/assets/clock-ejemplo-3-fixed.png)

Asimismo, el hecho que genere una carga de trabajo final de:
Worker 1: 8
Worker 2: 7
Worker 3: 6

contra 9 para el primer Worker y 6 para los restantes; demuestra que el algoritmo W-Clock de pre-planificación realiza una mejor distribución de la carga de trabajo que su contraparte que no contempla la carga de trabajo de los Workers.
