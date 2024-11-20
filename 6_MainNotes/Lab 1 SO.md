2024-11-06 12:35

__Status__: #Avanzada  
__Tags__: #FaMAF #SO #Lab #Computación #Kernel 
# Lab 1 SO

##### Volver a [[Sistemas Operativos]]

El laboratorio 1 de sistemas operativos consistió en implementar un interprete de consola llamada MyBash. En este
debíamos aprender a parsear lo introducido en consola, implementar  un TAD de comandos y pipes, y hacer un 
execute que se encargue de delegar las ejecuciones de los programas.

### Commands

Implementamos un TAD para hacer uso de comandos y pipelines. Se dio uso a dos estructuras, la primera fue :
 - command_s que tenía una lista enlazada del comando y los argumentos a modo de string, y dos strings más
que señalaban los files de input y output si los hubiese.
- pipeline_s daba uso a una lista enlazada de command_s y de un booleano que indicaba wait si el programa iba
a background.

El TAD debía ser opaco y encargarse de administrar la memoria que se le pasase.

### Parsing

El bloque parsing se encargaba de interpretar lo que escribía el usuario en consola y traducirlo a un pipeline. Pues
se usaba siempre un pipeline aunque fuera un solo comando el que se le pasase.

```c
typedef enum {
    ARG_NORMAL, // Indicates a command name or command argument type
    ARG_INPUT,  // Indicates an input redirection
    ARG_OUTPUT  // Indicates an output redirection
} arg_kind_t; // An auxiliary type for parser_next_argument() 
```
Estos enumerables se usaban para identificar qué se le pasase.

¿Cómo se parsea un comando?  Veamos un ejemplo:

MyBash $>  ls -l | wc -l

Esto se parsea de la siguiente manera:

( \[(\["ls", "-l"\], NULL, NULL), ( \["wc", "-l"\], NULL)], true)

donde el true indica que va a esperar y no se va a ira segundo plano.

### Execute

Este bloque es el encargado de manejar las ejecuciones de los programas usando las funciones fork, execvp, wait, dup, dup2, open,
close, write, read, pipe.
Repasemos qué hace cada función:

- fork(): Divide un proceso en dos, creando un hijo a partir del padre (el manejo de memoria se hace con copy on write), devuelve
un pid que sirve para identificar los procesos. Si el pid == 0 entonces es el hijo, si pid > 0 es el padre y si pid < 0 hubo un error.
- execvp(): de la familia de los exec. Ejecuta el programa que se le pase, recibe como argumento el nombre del programa, y 
un array de strings con los argumentos y el mismo programa. Toma total control del proceso, es decir, si se ejecuta en el hijo
del fork() se escapa del programa.
- wait: Espera que el hijo termine.
- dup: duplica el file descriptor que le pases en el primer lugar disponible.
- dup2: igual que dup, pero se le puede especificar a dónde copiarlo.
- open: abre un archivo y podés elegir con qué permisos o cómo abrirlo, devuelve un fd (file descriptor) en la primera posición
libre.
- close: Cierra un archivo a partir de su fd.
- write: Escribe en un fd que se le pase, dado un buffer y una cantidad de bytes.
- read: Lee de un fd que se le pase, dado un buffer y una cantidad de bytes.
- pipe: pipe recibe un arregle de tamaño dos e interconecta las entradas 0 y 1 formando una tubería para el paso de datos.
Lo que se escriba en 0 va a salir por 1.

### Main

Se llama a input parse para leer por consola y se entra a un loop que siempre va chequeando que le ingresa, lo parsea y lo 
ejecuta a poder ser. También está implementada con ctrl+D (signal EOF) para cerrar mybash.

### Test

Los test fueron implementados por los profes para chequear los distintos módulos. Veamos algunos ejemplos de sus 
outputs:

runner-command: command.c:214: pipeline_front: Assertion `self != NULL && !pipeline_is_empty(self)' failed.
runner-command: command.c:214: pipeline_front: Assertion `self != NULL && !pipeline_is_empty(self)' failed.
runner-command: command.c:221: pipeline_get_wait: Assertion `self != NULL' failed.
runner-command: command.c:226: pipeline_to_string: Assertion `self != NULL' failed.
100%: Checks: 48, Failures: 0, Errors: 0

Esto es un ejemplo de algo OK. Vamos uno no tan OK.

runner-command: command.c:24: scommand_new: Assertion `result != NULL && scommand_is_empty(result) && scommand_get_redir_in(result)==NULL && scommand_get_redir_out(result)==NULL' failed.
runner-command: command.c:24: scommand_new: Assertion `result != NULL && scommand_is_empty(result) && scommand_get_redir_in(result)==NULL && scommand_get_redir_out(result)==NULL' failed.
60%: Checks: 48, Failures: 0, Errors: 19
test_scommand.c:113:E:Creation:test_new_destroy:0: (after this point) Received signal 6 (Aborted)
test_scommand.c:121:E:Creation:test_new_is_empty:0: (after this point) Received signal 6 (Aborted)
(null):-1:S:Functionality:test_adding_emptying:0: (after this point) Received signal 6 (Aborted)
(null):-1:S:Functionality:test_adding_emptying_length:0: (after this point) Received signal 6 (Aborted)
(null):-1:S:Functionality:test_fifo:0: (after this point) Received signal 6 (Aborted)
(null):-1:S:Functionality:test_front_idempotent:0: (after this point) Received signal 6 (Aborted)
(null):-1:S:Functionality:test_front_is_back:0: (after this point) Received signal 6 (Aborted)
(null):-1:S:Functionality:test_front_is_not_back:0: (after this point) Received signal 6 (Aborted)
(null):-1:S:Functionality:test_redir:0: (after this point) Received signal 6 (Aborted)
(null):-1:S:Functionality:test_independent_redirs:0: (after this point) Received signal 6 (Aborted)
(null):-1:S:Functionality:test_to_string_empty:0: (after this point) Received signal 6 (Aborted)
(null):-1:S:Functionality:test_to_string:0: (after this point) Received signal 6 (Aborted)
test_pipeline.c:124:E:Functionality:test_adding_emptying:0: (after this point) Received signal 6 (Aborted)
test_pipeline.c:143:E:Functionality:test_adding_emptying_length:0: (after this point) Received signal 6 (Aborted)
test_pipeline.c:157:E:Functionality:test_fifo:0: (after this point) Received signal 6 (Aborted)
test_pipeline.c:176:E:Functionality:test_front_idempotent:0: (after this point) Received signal 6 (Aborted)
test_pipeline.c:188:E:Functionality:test_front_is_back:0: (after this point) Received signal 6 (Aborted)
test_pipeline.c:197:E:Functionality:test_front_is_not_back:0: (after this point) Received signal 6 (Aborted)
test_pipeline.c:229:E:Functionality:test_to_string:0: (after this point) Received signal 6 (Aborted)
make\[1]: *** \[Makefile:59: test-command] Error 1
make\[1]: se sale del directorio '/home/emiliano/Documentos/facultad/SisOP/labs/lab1/tests'
make: *** \[Makefile:39: test-command] Error 2

Ahora veamos los casos de los memtest.

\==81871== Command: ./leaktest
\==81871==
\==81871==
\==81871== HEAP SUMMARY:
\==81871==     in use at exit: 18,864 bytes in 13 blocks
\==81871==   total heap usage: 107 allocs, 94 frees, 20,345 bytes allocated
\==81871==
\==81871== 4 bytes in 1 blocks are definitely lost in loss record 2 of 13
\==81871==    at 0x4848899: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
\==81871==    by 0x4B3B58E: strdup (strdup.c:42)
\==81871==    by 0x10D2A7: scommand_memory_test (test_scommand.c:412)
\==81871==    by 0x10B895: main (leaktest.c:7)
\==81871==
\==81871== 8 bytes in 1 blocks are definitely lost in loss record 3 of 13
\==81871==    at 0x4848899: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
\==81871==    by 0x4B3B58E: strdup (strdup.c:42)
\==81871==    by 0x10D4DE: scommand_memory_test (test_scommand.c:444)
\==81871==    by 0x10B895: main (leaktest.c:7)
\==81871==
\==81871== LEAK SUMMARY:
\==81871==    definitely lost: 12 bytes in 2 blocks
\==81871==    indirectly lost: 0 bytes in 0 blocks
\==81871==      possibly lost: 0 bytes in 0 blocks
\==81871==    still reachable: 0 bytes in 0 blocks
\==81871==         suppressed: 18,852 bytes in 11 blocks
\==81871==
\==81871== For lists of detected and suppressed errors, rerun with: -s
\==81871== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 2 from 2)

Notar que este caso tengo 12 bytes perdidos que son memory leaks que no pudimos corregir, mientras que esos casi 19kbytes
suprimidos son de la librería glib.


## Referencias

-  [[Sistemas Operativos material]]
-  [[Lab1 - Enunciado.pdf]]
