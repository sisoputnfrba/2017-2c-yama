# Descripción de las entregas

## Hito 1: Checkpoint 1

- **Fecha**: 16 de Septiembre
- **Timepo Estimado**: 2 semanas
- **Objetivos**:
  - Familiarizarse con Linux y su consola, el entorno de desarrollo y el repositorio
  - Aplicar las Commons Libraries, principalmente las funciones para listas, archivos de conf y logs
  - Definir el Protocolo de Comunicación
  - Implementar la consola del FileSystem sin funcionalidades
  - Desarrollar una comunicación simple entre los procesos que permita propagar un archivo abierto por Master.
  - Familiarizarse con el desarrollo de procesos servidor multihilo (proceso Master) y server con forks(Worker)
- Lectura recomendada:
  - [http://faq.utnso.com/arrancar](http://faq.utnso.com/arrancar)
  - [Beej Guide to Network Programming](http://beej.us/guide/bgnet/output/html/multipage/index.html)
  - [Linux POSIX Threads](http://www.yolinux.com/TUTORIALS/LinuxTutorialPosixThreads.html)
  - [SO UTN FRBA Commons Libraries](https://github.com/sisoputnfrba/so-commons-library)
  - Sistemas Operativos, Silberschatz, Galvin - Capítulo 4: Hilos

## Hito 2: Avance del Grupo[8]

- **Fecha**: 7 de Octubre
- **Timepo Estimado**: 3 semanas
- **Objetivos**:
  - Implementación de la base del Protocolo de Comunicación
  - Comprender y aplicar `mmap()`
  - Desarrollar las  Estructuras del FileSystem (Tablas de Directorios, Archivos, Nodos, Bitmap)
  - YAMA debe poder planificar tareas mediante Round Robin
  - Master y YAMA pueden efectuar todo un recorrido (sin comunicación contra el Worker ni FS) respetando el RR
  - El FileSystem debe permitir la copa de un archivo al yamafs y almacenarlo partiéndolo en bloques.
- Lectura recomendada:
  - Sistemas Operativos, Silberschatz, Galvin - Capítulo 4: Hilos
  - Sistemas Operativos, Silberschatz, Galvin - Capítulo 10 y 11: Sistemas de Archivos
  - Sistemas Operativos, Silberschatz, Galvin - Capítulo 17:  Sistemas de Archivos Distribuidos

## Hito 3: Checkpoint Presencial en el Laboratorio

- **Fecha**: 21 de Octubre
- **Tiempo Estimado**: 2 semanas
- **Objetivos**:
  - Los procesos Master y YAMA manejan errores durante la ejecución de un Job.
  - El proceso YAMA puede replanificar una etapa de planificación y admite planificación con WRR.
  - El proceso YAMA debe poder conectarse al FileSystem y levantar la metadata sobre un archivo sobre el cual ejecutar un Job.
  - La Consola del FileSystem es capaz de reproducir las operaciones de copia de archivos contra el FileSystem local y la generación del md5 de un archivo almacenado.
  - Los procesos Worker deben poder recibir archivos por socket y darles permisos de ejecución.
- Lectura recomendada:
  - [Ejemplos de implementación de redirección de stdin/stdout mediante pipes](https://github.com/nicozar95/ejemplo_correr-script_so)
  - [Cómo realizar una consola interactiva](https://goo.gl/XU675H)

## Hito 4: Entrega Final  

- **Fecha**: 25 de Noviembre
- **Objetivos**:
  - El proceso Master calcula las métricas de un Job
  - La Consola del FileSystem es capaz de efectuar operaciones sobre Nodos y Bloques.
  - El proceso YAMA maneja correctamente la señal SIGUSR1 y recarga su configuración.
  - Es posible realizar una inicialización del FileSystem con datos viejos.
  - Todos los componentes del TP ejecutan los requerimientos pedidos  de forma integral, bajo escenarios de stress.
- Lectura recomendada:
  - [Guías de Debugging del Blog utnso.com](https://www.utnso.com/recursos/guias/)
  - [MarioBash: Tutorial para aprender a usar la consola](https://www.utnso.com/recursos/guias/)

## Otras entregas:
- Segunda entrega:        2 de Diciembre
- Tercera entrega:        16 de Diciembre

[8] No es requerido entregar nada, es a título informativo
