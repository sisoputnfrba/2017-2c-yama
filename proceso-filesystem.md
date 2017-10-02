# Proceso FileSystem

Este proceso será el encargado de organizar los bloques de datos distribuidos en los procesos DataNode de cada Nodo, asociarlos a los correspondientes nombres de archivos para así exponer el sistema de archivos a los usuarios.

Al iniciar, leerá su archivo de configuración, si lo tuviese, y quedará a la espera de las conexiones de los procesos DataNode. Una vez que se hayan conectado los DataNode requeridos, se deberá formatear el FileSystem, pasando el mismo a un **estado estable**[^2], en el cual permitirá que se conecten Workers o YAMA, a quien responderá solicitudes sobre los archivos. Mientras que el FileSystem no adquiera un estado estable, el mismo no deberá permitir la conexión de YAMA u otro proceso que no sean DataNode. Además, una vez formateado el FileSystem, no se permitirán nuevos Nodos en el sistema.

## Estructura del FileSystem

El diseño de las estructuras está orientado a flujos de datos WORM (write once, read many), en otras palabras, para situaciones donde los datos son leídos frecuentemente pero ocasionalmente (o nunca) son modificados.

Adicionalmente tiene la particularidad de que almacena una copia de cada bloque de datos en otro nodo para así lograr la redundancia en caso de que un Nodo salga de línea abruptamente.

La asignación de cada copia de un bloque en un nodo se realizará de forma **contigua**. Es decir, se asignará el espacio dentro del Nodo al primer espacio libre, siguiendo los lineamientos del algoritmo First Fit.

A continuación se mostrará un ejemplo de asignación de los espacios para los bloques de un archivo de ejemplo. Como aclaración, se deberá asumir que la imagen es solamente de carácter ilustrativo.
![Ejemplo de File system](/assets/filesystem-ejemplo.png)

_Como se observa en el diagrama, el archivo ejemplo.csv, compuesto por 3 bloques, está distribuido en los Nodos 1, 2 y 3 teniendo dos copias de cada bloque. Puede ver las estructuras administrativas para este archivo `ejemplo.csv` en: [Esquema de filesystem de ejemplo](proceso-filesystem.md#esquema-de-filesystem-de-ejemplo)._

## Bloques

El tamaño de bloque es fijo de **1MB** y con la particularidad de que este filesystem permite diferenciar entre archivos de texto o binarios.

En el caso de los archivos de texto, dado que los mismos pueden no necesariamente terminan alineados a 1 MB, para evitar que queden registros partidos la operación de escritura debe asegurarse antes de escribir cada nuevo renglón (`\n`)[^3] que el espacio en el bloque sea suficiente. En caso de que no lo sea, anotará la posición del último byte con contenido válido en la estructura administrativa del archivo y continuará en el próximo bloque. Cabe aclarar, que incluso si un bloque no estuviera completamente ocupado, solamente podrá ser ocupado por un único archivo a la vez.

En el caso de los archivos binarios, todo el bloque debe utilizarse completo.

El proceso FileSystem debe en todo momento conocer la cantidad de bloques libres y utilizados de cada Nodo y por consiguiente **el espacio libre total**[^4].
Al recibir una operación de escritura deberá balancear la asignación de los bloques entre todos los Nodos del sistema, junto con evitar que el mismo bloque de datos de un archivo esté más de una vez en el mismo nodo.


## Esquema de filesystem de ejemplo

- `ejemplo.csv`
  - Tamaño: 3.145.592 bytes
  - Tipo: Delimitado
  - Directorio Padre: 5
  - Estado: Disponible
  - Bloques:


  | Bloque | Copia 1            | Copia 2           | Fin Bloque |
  |--------|--------------------|-------------------|------------|
  | 0      | Nodo 1 - Bloque 5  | Nodo 2 - Bloque 2 | 1048576    |
  | 1      | Nodo 2 - Bloque 10 | Nodo 3 - Bloque 7 | 1048500    |
  | 2      | Nodo 1 - Bloque 12 | Nodo 2 - Bloque 3 | 1048516    |


- `foto.jpg`
  - Tamaño: 1.472.461 bytes
  - Tipo: Binario
  - Directorio Padre: 1
  - Estado: Disponible
  - Bloques:


  | Bloque | Copia 1           | Copia 2           | Fin Bloque |
  |--------|-------------------|-------------------|------------|
  | 0      | Nodo1 - Bloque 43 | Nodo2 - Bloque 18 | 1048576    |
  | 1      | Nodo3 - Bloque 56 | Nodo4 - Bloque 66 | 1048576    |
  | 2      | Nodo4 - Bloque 91 | Nodo2 - Bloque 75 | 1048576    |
  | ...    | ..                | ..                |            |


## Proceso de Inicialización

Al iniciar el proceso Filesystem y los diversos procesos DataNode el sistema debe poder restaurarse de un estado anterior, si el mismo existiese. Para ello, el proceso FileSystem persiste una suerte de archivos que mantienen las estructuras almacenadas en disco.
Si existe un estado anterior (es decir, existiesen todos los archivos mencionados), el proceso FileSystem levantará todas las estructuras y quedará a la espera de los procesos DataNode.

Durante el establecimiento de la conexión, el FileSystem comparará el id del Nodo al que pertenece el DataNode, y si el mismo corresponde a un Nodo de un estado anterior, el mismo será agregado como válido, asumiendo que los datos de los bloques son correctos.

Mientras que el FileSystem no posea al menos 1 copia de cada uno de los archivos almacenados en los DataNodes conectados, se denomina que **el proceso FileSystem está en un estado no-estable**. Una vez adquiridas al menos 1 copia de cada archivo, se considerará que **el proceso FileSystem ha pasado a un estado estable** y permitirá nuevamente conexiones de los demás procesos. Además, aunque el proceso FileSystem pase a un _estado estable_, permitirá que los nodos que todavía no se conectaron, vuelvan a hacerlo.

Como aclaración final, si el proceso FileSystem es ejecutado con el flag `--clean` se procederá a ignorar e eliminar el estado anterior.

Ejemplo de ejecución de un proceso FileSystem ignorando el estado anterior: `./yamafs --clean`

## Interfaz del proceso Filesystem

Contará con una interfaz detallada contra los Nodos, la cual no puede ser modificada ni ampliada pero la definición del protocolo será definida por el grupo.

### Almacenar archivo

Recibirá una ruta completa, el nombre del archivo, el tipo (texto o binario) y los datos correspondientes. Responderá con un mensaje confirmando el resultado de la operación.
Es responsabilidad del proceso Filesystem:
1. “Cortar” el archivo en registros completos hasta 1MB en el caso de los archivos de texto o en bloques de 1 MB en el caso de los binarios.
2. Designar el DataNode y el Bloque que va a almacenar cada copia[^5] utilizando sus estructuras administrativas y balanceando el contenido de manera homogénea entre los diversos DataNodes disponibles.
3. Enviar el contenido del bloque a los DataNodes designados.

### Leer archivo

Recibirá una ruta completa con el nombre del archivo. Responderá con el contenido completo del mismo.
Es responsabilidad del proceso Filesystem:
1. Solicitar a los diferentes DataNodes los bloques correspondientes al archivo considerando que las solicitudes deberán distribuirse **de forma equitativa** en los diferentes DataNodes según cómo estén distribuidas las copias de los bloques.


### Consola del Proceso Filesystem

Este proceso tendrá una consola desde la cual se podrán ejecutar comandos de administración de archivos. Se recomienda la utilización de la biblioteca readline[^6] para la captura de teclas especiales.

A continuación se hará una breve descripción de los comandos a utilizar mediante una sintaxis cómoda pensado por la cátedra. La sintaxis de los mismos **no es de carácter obligatorio**, pero se recomienda su uso dada la similaridad con los comandos usuales.
Como aclaración, algunos comandos pueden llegar a ser reutilizables para diferentes tareas. Debido a esto, cada una de ellas tendrán asignadas un flag (no combinables entre sí) para identificar de qué comportamiento se trata.
1. `format` - Formatear el Filesystem.
2. `rm [path_archivo]` ó `rm -d  [path_directorio]` ó `rm -b  [path_archivo] [nro_bloque] [nro_copia]` - Eliminar un Archivo/Directorio/Bloque. Si un directorio a eliminar no se encuentra vacío, la operación debe fallar. Además, si el bloque a eliminar fuera la última copia del mismo, se deberá abortar la operación informando lo sucedido.
3. `rename [path_original] [nombre_final]` - Renombra un Archivo o Directorio
4. `mv [path_original] [path_final]` - Mueve un Archivo o Directorio
5. `cat [path_archivo]` - Muestra el contenido del archivo como texto plano.
6. `mkdir [path_dir]` - Crea un directorio. Si el directorio ya existe, el comando deberá informarlo.
7. `cpfrom [path_archivo_origen] [directorio_yamafs]` - Copiar un archivo local al yamafs, siguiendo los lineamientos en la operaciòn Almacenar Archivo, de la Interfaz del FileSystem.
8. `cpto [path_archivo_yamafs] [directorio_filesystem]` - Copiar un archivo local desde el yamafs
9. `cpblock [path_archivo] [nro_bloque] [id_nodo]` - Crea una copia de un  bloque de un archivo en el nodo dado.
10. `md5 [path_archivo_yamafs]` - Solicitar el MD5 de un archivo en yamafs
11. `ls [path_directorio]` - Lista los archivos de un directorio
12. `info [path_archivo]` - Muestra toda la información del archivo, incluyendo tamaño, bloques, ubicación de los bloques, etc.

## Estructuras del proceso filesystem

Las distintas estructuras administrativas del proceso Filesystem se persisten en archivos individuales respetando el siguiente formato

### Tabla de Directorios

El Filesystem soporta directorios hasta un total de 100 directorios. Todos los archivos y directorios deben tener un directorio padre, el cual puede ser otro directorio o el directorio raíz (padre=0).

| Index | Directorio | Padre |
|-------|------------|-------|
| 0     | root       | -1    |
| 1     | user       | 0     |
| 2     | jose       | 1     |
| 3     | juan       | 1     |
| 4     | temporal   | 0     |
| 5     | datos      | 2     |
| 6     | fotos      | 2     |
| 99    | ...        | ...   |

Como pueden ver este esquema permite el siguiente diagrama:

```
raiz (/)
|--- /user
|     |--- /user/juan
|     |    |--- /user/juan/datos
|     |    |--- /user/juan/fotos
|     |--- /user/jose
|--- /temporal
```

La tabla de directorios debe ser almacenada en un archivo en el filesystem local (no yamafs) denominado metadata/directorios.dat como un array de 100 posiciones de `struct t_directory`.

```
struct t_directory {
  int index,
  char nombre[255],
  int padre
}
```

## Tabla de archivos

La información de cada archivo dentro de yamafs será almacenada en archivo texto compatible dentro de un directorio cuyo nombre corresponde al índice de la tabla de directorios.
Siguiendo con el ejemplo anterior todos los archivos de `/user/juan/datos serán almacenados en metadata/archivos/5/`.

```
metadata/archivos/5/ejemplo.csv
TAMANIO=12356334
TIPO=TEXTO
BLOQUE0COPIA0=[Nodo1, 33]
BLOQUE0COPIA1=[Nodo2, 11]
BLOQUE0BYTES=1048500
BLOQUE1COPIA0=[Nodo1, 34]
BLOQUE1COPIA1=[Nodo2, 12]
BLOQUE0BYTES=1048532
...
```

## Tabla de Nodos

Iniciar el filesystem, el mismo deberá registrar los DataNodes que se vayan conectando y terminan componiendo dicho sistema de archivos. Además, deberá registrar su cantidad de bloques total y libres. Esto se persistirá en el `archivo metadata/nodos.bin`.

```
TAMANIO=300
LIBRE=171
NODOS=[Nodo1, Nodo2, Nodo3]
Nodo1Total=50
Nodo1Libre=16
Nodo2Total=100
Nodo2Libre=80
Nodo3Total=150
Nodo3Libre=75
```

## Bitmap de Bloques por Nodo

Por cada DataNode del Filesystem se deberá almacenar un bitmap indicando el estado (libre/ocupado) de cada bloque.
Esta información se deberá persistir en `metadata/bitmaps/[nombre-de-nodo].dat`.
Ejemplo: `metadata/bitmaps/nodo2.dat`


### Archivo de Logs

El proceso FileSystem deberá registrar toda su actividad mediante un archivo de log, sin mostrar por pantalla dichos logs, por la existencia de una consola. Sin embargo, sí se deberán mostrar por pantalla los resultados de las tareas realizadas mediante la consola.

[^2]: Ver apartado sobre Proceso de Inicialización
[^3]: El valor `\n` indica ‘New Line’ o ‘Nueva línea’. Equivale al ingreso de un `[Enter]` en el teclado.
[^4]: El alumno debe investigar opciones para almacenar esta información de manera eficiente
[^5]: No está permitido almacenar las dos copias en el mismo DataNode
[^6]: Se recomienda la lectura previa del tutorial [Cómo realizar una consola interactiva](https://goo.gl/XU675H)
