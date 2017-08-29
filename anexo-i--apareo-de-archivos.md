# Anexo I - Apareo de Archivos

Apareo es el nombre que se le da a una rutina que dado N archivos ordenados genera uno nuevo con el resultado ordenado de la unión de los mismos.

En términos de procesamiento, realizar el concatenado de los N archivos y luego ordenarlo resulta muy costoso, por lo que teniendo en cuenta que los archivos de origen ya están ordenados se realiza comparando registro por registro.

Como puede observar en el ejemplo en una primera etapa se comparan los primeros registros de cada archivo (Ana, Belen y Angeles).
Dado que Ana es el menor alfabéticamente este es trasladado al archivo de resultado.
Luego leemos el segundo nombre del primer archivo (Marta) y comparamos nuevamente.

![Apareo de archivos](/assets/apareo.png)
