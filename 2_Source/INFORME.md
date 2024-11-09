## Integrantes:
* Agustin Cosano
* Eliseo Facchin
* Emiliano Guardabassi
* Santiago Zuk

---

# Primera parte

1. ¿Qué política de planificación utiliza 'xv6-riscv' para elegir el próximo 
proceso a ejecutarse?

Xv6 utiliza una política de Round Robin (RR) para elegir el próximo proceso a 
ejecutarse. Xv6 mantiene una lista de procesos que están en estado RUNNABLE y de
ahí selecciona el próximo a ejecutar.

2. ¿Cuáles son los estados en los que un proceso puede permanecer en xv6-riscv y
qué los hace cambiar de estado?

Los estados en los que un proceso puede estar son: 
UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE.

* __UNUSED__: El proceso no está en uso y su estructura está libre. Cuando se 
llama a allocproc() pasa a USED, cuando se llama a fork() pasa a RUNNABLE.

* __USED__: El proceso está en uso, pero no necesariamente para ejecutarse. Este
estado se establece cuando se asigna un nuevo proceso, pero aún no está en 
condiciones de correr.

* __SLEEPING__: El proceso está bloqueado esperando una llamada de wakeup() para
pasar a RUNNABLE.

* __RUNNABLE__: El proceso está listo para ejecutarse pero está esperando ser 
elegido por el scheduler.

* __RUNNING__: El proceso se está ejecutando en este momento. yield() lo hace 
pasar a estado RUNNABLE. sleep() lo hace pasar de RUNNING a SLEEPING.

* __ZOMBIE__: Cuando se ejecuta exit() el proceso para a este estado hasta que 
su padre llame a wait(), entonces pasa a UNUSED.

3. ¿Qué es un *quantum*?¿Dónde se define en el código?¿Cuánto dura un *quantum* 
en xv6-riscv?

Un *quantum* es un intervalo fijo de tiempo en el que un proceso puede estar 
ejecutandose en el cpu antes de que el sistema operativo lo interrumpa para 
cambiar de proceso. En xv6 está definido en el archivo kernel/start.c:69 en la 
función timerinit(). Un *quantum* en este caso dura 1000000 ciclos de cpu, como 
1/10 partes de segundo en qemu.

4. ¿En qué parte del código ocurre el cambio de contexto en xv6-riscv? ¿En qué 
funciones un proceso deja de ser ejecutrado? ¿En qué funciones se elige el nuevo
proceso a ejecutar?

El cambio de contexto ocurre en la función swtch() que guarda los registros del 
proceso que estaba ejecutandose y carga los registros del proceso que va a 
ejecutandose.Un proceso deja de ser ejecutado cuando se llaman las funciones 
yield() que cede el cpu, sleep() que se llama normalmente cuando hay un I/O y 
bloquea el programa, exit() que termina el programa y cede el cpu.

En la función scheduler() recorre una lista de procesos y el primero que 
encuentre en estado RUNNABLE va a ser ejecutando en la proxima interrupción. 
Una vez seleccionado el proceso le llama a swtch() que realiza el context switch 
para ejecutar el nuevo proceso.

5. ¿El cambio de contexto consume tiempo de un *quantum*?

Sí, si consume tiempo del *quantum*. Durante un cambio de contexto, el *SO* debe
guardar el estado del proceso actual,
y restaurar el estado del próximo proceso que se va a ejecutar. Esto implica 
operaciones que, aunque sean rápidas, requieren algo de tiempo para completarse.

---

# Segunda parte

#### specs del pc utilizado para experimento 1:

| Hardware/Software |                                  |
| ----------------- | -------------------------------- |
| OS:               | Linux MInt 21.3 x86_64           |
| CPU:              | AMD Ryzen 5 3400G (8) @ 3.699GHz |
| GPU:              | AMD ATI Radeon RX 6700           |
| RAM:              | 32032 MiB 2667 MT/s              |


## Experimento 1 

#### Valores elegidos para el experimento:

* Tamaño del N: Se decidió usar un N igual a 15 que da lugar a cambios de 
contexto  pero no extiende demasiado el experimento.

* Tamaño del *quantum*: cómo pedía la consigna el *quantum* usado es 10 veces 
más chico que el original.

* Métrica o métricas: Elegimos en este caso una métrica distinta para cpubench 
de la de iobench, aunque son conceptualmente iguales. 

1. Para cpubench usamos:
```
total_cpu_kops/elapsed_ticks; 
```
Que toma el total de operaciones hechas y las divide por el tiempo de ejecución.

2. Para iobench usamos :
```
(total_iops * 100)/elapsed_ticks; 
```
Que toma el total de operaciones hechas y las divide por el tiempo de ejecución,
la diferencia es que las operaciones están multiplicadas por 100 dado que los 
resultados eran muy chicos en los experimentos que tenían algún tipo de starving.

### Respuestas

1. Los parámetros usados  en los programas cpubench e iobench son los siguientes
(métrica ya explicada):

    * cpubench:

        ```
        CPU_MATRIX_SIZE 128
        CPU_EXPERIMENT_LEN 256
        MEASURE_PERIOD 1000
        N 15
        ```
    
Los valores son los que estaban originalmente en el programa, salvo el N que
lo elegimos nosotros. Los nombres son bastante descriptivos, pero vamos a 
explicar de todas formas qué significan. CPU_MATRIX_SIZE es el tamaño de las
matrices que el programa va a multiplicar, CPU_EXPERIMENT_LEN es la cantidad
de veces que se va a hacer esta multiplicación y MEASURE_PERIOD es el 
intervalo de medición pero no se usa en esta implementación.

    * iobench:
        
        ```
        IO_OPSIZE 64
        IO_EXPERIMENT_LEN 512
        N 15
        ```

Los valores son los que estaban originalmente en el programa, salvo el N que
lo elegimos nosotros. IO_OPSIZE indica el tamaño de las operaciones IO, 
define el tamaño del buffer ```static char data[IO_OPSIZE];``` y se usa de 
argumento en read() y write(). IO_EXPERIMENT_LEN es la cantidad de veces que
se va a hacer el llamado a operaciones de lectura y escritura.
<br>
2. ¿Los procesos se ejecutan en paralelo? ¿En promedio, qué proceso o procesos 
se ejecutan primero?

Para responder esta pregunta tomemos como ejemplo el experimento:
```iobench 15 &; cpubench 15 &; cpubench 15 &; cpubench 15 &```
que da como resultado las siguientes tablas:

| pid | type     |         | metric | start_tick | elapsed_ticks |
|:---:|:--------:|:-------:|:------:|:----------:|:-------------:|
| 5  | [iobench] |io_metric | 41     | 235        | 2476          |
| 5  | [iobench] |io_metric | 32     | 2711       | 3115          |
| 5  | [iobench] |io_metric | 253    | 5826       | 404           |
| 5  | [iobench] |io_metric | 640    | 6230       | 160           |
| 5  | [iobench] |io_metric | 644    | 6391       | 159           |
| 5  | [iobench] |io_metric | 644    | 6551       | 159           |
| 5  | [iobench] |io_metric | 644    | 6711       | 159           |
| 5  | [iobench] |io_metric | 648    | 6871       | 158           |
| 5  | [iobench] |io_metric | 636    | 7030       | 161           |
| 5  | [iobench] |io_metric | 640    | 7192       | 160           |
| 5  | [iobench] |io_metric | 640    | 7352       | 160           |
| 5  | [iobench] |io_metric | 640    | 7512       | 160           |
| 5  | [iobench] |io_metric | 652    | 7673       | 157           |
| 5  | [iobench] |io_metric | 648    | 7831       | 158           |
| 5  | [iobench] |io_metric | 660    | 7989       | 155           |
|---|------------|---------- -|------|------|-----|
| 7 | [cpubench] | cpu_metric | 1588 | 282  | 338 |
| 7 | [cpubench] | cpu_metric | 2294 | 849  | 234 |
| 7 | [cpubench] | cpu_metric | 2473 | 1242 | 217 |
| 7 | [cpubench] | cpu_metric | 2462 | 1630 | 218 |
| 7 | [cpubench] | cpu_metric | 2462 | 2017 | 218 |
| 7 | [cpubench] | cpu_metric | 2840 | 2404 | 189 |
| 7 | [cpubench] | cpu_metric | 1439 | 2710 | 373 |
| 7 | [cpubench] | cpu_metric | 1760 | 3204 | 305 |
| 7 | [cpubench] | cpu_metric | 1412 | 3617 | 380 |
| 7 | [cpubench] | cpu_metric | 1988 | 4003 | 270 |
| 7 | [cpubench] | cpu_metric | 1737 | 4359 | 309 |
| 7 | [cpubench] | cpu_metric | 2508 | 4747 | 214 |
| 7 | [cpubench] | cpu_metric | 2265 | 5116 | 237 |
| 7 | [cpubench] | cpu_metric | 2072 | 5561 | 259 |
| 7 | [cpubench] | cpu_metric | 3578 | 5941 | 150 |
|---|------------|------------|------|------|-----|
| 9 | [cpubench] | cpu_metric | 1516 | 238  | 354 |
| 9 | [cpubench] | cpu_metric | 2855 | 627  | 188 |
| 9 | [cpubench] | cpu_metric | 1777 | 991  | 302 |
| 9 | [cpubench] | cpu_metric | 1813 | 1382 | 296 |
| 9 | [cpubench] | cpu_metric | 1813 | 1768 | 296 |
| 9 | [cpubench] | cpu_metric | 1801 | 2155 | 298 |
| 9 | [cpubench] | cpu_metric | 1512 | 2734 | 355 |
| 9 | [cpubench] | cpu_metric | 3103 | 3095 | 173 |
| 9 | [cpubench] | cpu_metric | 2933 | 3431 | 183 |
| 9 | [cpubench] | cpu_metric | 1454 | 3620 | 369 |
| 9 | [cpubench] | cpu_metric | 2097 | 4006 | 256 |
| 9 | [cpubench] | cpu_metric | 1838 | 4363 | 292 |
| 9 | [cpubench] | cpu_metric | 1754 | 4878 | 306 |
| 9 | [cpubench] | cpu_metric | 1896 | 5264 | 283 |
| 9 | [cpubench] | cpu_metric | 2182 | 5564 | 246 |
|----|------------|------------|------|------|-----|
| 10 | [cpubench] | cpu_metric | 1512 | 239  | 355 |
| 10 | [cpubench] | cpu_metric | 2407 | 702  | 223 |
| 10 | [cpubench] | cpu_metric | 1844 | 995  | 291 |
| 10 | [cpubench] | cpu_metric | 1883 | 1386 | 285 |
| 10 | [cpubench] | cpu_metric | 1857 | 1772 | 289 |
| 10 | [cpubench] | cpu_metric | 1857 | 2159 | 289 |
| 10 | [cpubench] | cpu_metric | 3253 | 2544 | 165 |
| 10 | [cpubench] | cpu_metric | 1478 | 2714 | 363 |
| 10 | [cpubench] | cpu_metric | 1819 | 3206 | 295 |
| 10 | [cpubench] | cpu_metric | 1443 | 3621 | 372 |
| 10 | [cpubench] | cpu_metric | 2532 | 4236 | 212 |
| 10 | [cpubench] | cpu_metric | 2738 | 4605 | 196 |
| 10 | [cpubench] | cpu_metric | 1795 | 4882 | 299 |
| 10 | [cpubench] | cpu_metric | 1938 | 5268 | 277 |
| 10 | [cpubench] | cpu_metric | 3531 | 5826 | 152 |

Los procesos van turnandose a medida que consumen su tiempo de ejecucion, por 
ello en una llamada de cpubench o iobench puede haber una gran variacion en 
los tiempos de inicio de cada ciclo del bench; por ejemplo, en el segundo 
cpubench, en el cual el primer ciclo inicia en 282, hasta que este termina de
ejecutarse y se llega al segundo ciclo, entre medio se ejecutan otros procesos.
Con respecto a la prioridad de ejecucion, no tiende a tener preponderancia 
ningun proceso sobre el otro pero si iobench inicia con la ejecucion, este no la
termina, pasa al estado SLEEPING y debido a los tiempos de espera al usuario de 
los procesos iobound, la ejecucion de los procesos cpubench termina primero y 
luego continuan corriendo los procesos io hasta su terminacion. Esto ultimo 
puede verse en tabla, con el proceso iobench siendo el primero que inicia pero
tambien el que, por lo menos los 3 primeros ciclos, mas tarda en completarse.

<br>
3. ¿Cambia el rendimiento de los procesos iobound con respecto a la cantidad y 
tipos de procesos que se esten ejecutando en paralelo? ¿por qué?

Sí, cambia el rendimiento, y se comporta distinto si lo que se está ejecutando 
en paralelo es otro iobench o un cpubench.

##### Casos:

* __cpubench__: Para este caso podémos repasar la tabla del punto 2, donde se 
ejecuta un sólo iobench con tres cpubench en simultaneo. Vemos cómo el proceso
iobound arranca primero en le _tick 235_ pero rápidamente es acaparado por los
procesos cpubound y los iobound quedan esperando su turno en una suerte de 
_"starving"_. Una vez los cpubound concluyen, el iobound vuelve a comportarse
con normalidad, cómo si estuviera ejecutandose en solitario (con métricas 
sin casi variación entre ellas).

* __iobench__: En este caso utilizaremos el resultado del experimeto:
``` iobench 15 &; iobench 15 &; iobench 15 &```

| pid  | type      |           | metric | start_tick | elapsed_ticks |
|:----:|:---------:|:---------:|:------:|:----------:|:-------------:|
| 5    | [iobench] | io_metric | 489    | 418        | 209           |
| 5    | [iobench] | io_metric | 656    | 638        | 156           |
| 5    | [iobench] | io_metric | 793    | 795        | 129           |
| 5    | [iobench] | io_metric | 562    | 927        | 182           |
| 5    | [iobench] | io_metric | 867    | 1117       | 118           |
| 5    | [iobench] | io_metric | 846    | 1237       | 121           |
| 5    | [iobench] | io_metric | 793    | 1370       | 129           |
| 5    | [iobench] | io_metric | 966    | 1510       | 106           |
| 5    | [iobench] | io_metric | 939    | 1630       | 109           |
| 5    | [iobench] | io_metric | 556    | 1761       | 184           |
| 5    | [iobench] | io_metric | 1034   | 1973       | 99            |
| 5    | [iobench] | io_metric | 439    | 2075       | 233           |
| 5    | [iobench] | io_metric | 957    | 2323       | 107           |
| 5    | [iobench] | io_metric | 806    | 2431       | 127           |
| 5    | [iobench] | io_metric | 706    | 2561       | 145           |
|------|-----------|-----------|--------|------------|---------------|
| 7    | [iobench] | io_metric | 664    | 417        | 154           |
| 7    | [iobench] | io_metric | 706    | 575        | 145           |
| 7    | [iobench] | io_metric | 930    | 722        | 110           |
| 7    | [iobench] | io_metric | 544    | 835        | 188           |
| 7    | [iobench] | io_metric | 624    | 1030       | 164           |
| 7    | [iobench] | io_metric | 825    | 1195       | 124           |
| 7    | [iobench] | io_metric | 793    | 1320       | 129           |
| 7    | [iobench] | io_metric | 492    | 1454       | 208           |
| 7    | [iobench] | io_metric | 800    | 1664       | 128           |
| 7    | [iobench] | io_metric | 673    | 1793       | 152           |
| 7    | [iobench] | io_metric | 706    | 1953       | 145           |
| 7    | [iobench] | io_metric | 1101   | 2100       | 93            |
| 7    | [iobench] | io_metric | 781    | 2195       | 131           |
| 7    | [iobench] | io_metric | 800    | 2338       | 128           |
| 7    | [iobench] | io_metric | 846    | 2467       | 121           |
|------|-----------|-----------|--------|------------|---------------|
| 8    | [iobench] | io_metric | 914    | 418        | 112           |
| 8    | [iobench] | io_metric | 752    | 534        | 136           |
| 8    | [iobench] | io_metric | 656    | 685        | 156           |
| 8    | [iobench] | io_metric | 1024   | 842        | 100           |
| 8    | [iobench] | io_metric | 906    | 950        | 113           |
| 8    | [iobench] | io_metric | 860    | 1075       | 119           |
| 8    | [iobench] | io_metric | 721    | 1196       | 142           |
| 8    | [iobench] | io_metric | 644    | 1340       | 159           |
| 8    | [iobench] | io_metric | 890    | 1501       | 115           |
| 8    | [iobench] | io_metric | 494    | 1617       | 207           |
| 8    | [iobench] | io_metric | 1044   | 1825       | 98            |
| 8    | [iobench] | io_metric | 691    | 1924       | 148           |
| 8    | [iobench] | io_metric | 706    | 2075       | 145           |
| 8    | [iobench] | io_metric | 731    | 2221       | 140           |
| 8    | [iobench] | io_metric | 752    | 2369       | 136           |

Notemos que en este caso la métrica nos dá con una variación importante en todas
sus ocurrencias. Para tener noción de la magnitud de esto veamos una tabla de un
proceso iobound aislado: ```iobench 15 &```

| pid | type      |           | metric | start_tick | elapsed_ticks |
|:---:|:---------:|:---------:|:------:|:----------:|:-------------:|
| 4   | [iobench] | io_metric | 636    | 728        | 161           |
| 4   | [iobench] | io_metric | 660    | 889        | 155           |
| 4   | [iobench] | io_metric | 656    | 1045       | 156           |
| 4   | [iobench] | io_metric | 656    | 1202       | 156           |
| 4   | [iobench] | io_metric | 648    | 1358       | 158           |
| 4   | [iobench] | io_metric | 644    | 1516       | 159           |
| 4   | [iobench] | io_metric | 660    | 1675       | 155           |
| 4   | [iobench] | io_metric | 656    | 1831       | 156           |
| 4   | [iobench] | io_metric | 656    | 1987       | 156           |
| 4   | [iobench] | io_metric | 648    | 2143       | 158           |
| 4   | [iobench] | io_metric | 648    | 2301       | 158           |
| 4   | [iobench] | io_metric | 648    | 2460       | 158           |
| 4   | [iobench] | io_metric | 644    | 2618       | 159           |
| 4   | [iobench] | io_metric | 632    | 2777       | 162           |
| 4   | [iobench] | io_metric | 656    | 2940       | 156           |

El rendimiento en este caso se ve afectado haciendo que algunas operaciones 
tomen más tiempo para ejecutarse mientras que otras menos. Si bien no estamos
seguros exactamente de por qué sucede esto, podemos hipotetizar.
Los resultados de metrica más pequeña implican que el proceso demoró más _ticks_
en ser completado, esto se puede deber a los _context switches_ que ocurren al
ejecutar los programas en paralelo. Mientras que los casos con métrica más
grande implican que el preoceso se ejecutó en menos _ticks_, esto puede deberse
a que el _buffer_ ya estaba cargado de un proceso anterior y por lo tanto su 
ejecución fue más eficiente.

<br>
4. ¿Cambia el rendimiento de los procesos cpubound con respecto a la cantidad y
tipo de procesos que se estén ejecutando en paralelo? ¿Por qué?

``` cpubench 15 & ```

| pid | type       |                 | metric | start_tick | elapsed_ticks |
|:---:|:----------:|:---------------:|:------:|:----------:|:-------------:|
| 4   | [cpubench] | cpu_metric | 4329   | 7789       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 7914       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 8038       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 8163       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 8287       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 8412       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 8536       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 8661       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 8785       | 124           |
| 4   | [cpubench] | cpu_metric | 4364   | 8910       | 123           |
| 4   | [cpubench] | cpu_metric | 4329   | 9034       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 9159       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 9283       | 124           |
| 4   | [cpubench] | cpu_metric | 4329   | 9408       | 124           |
| 4   | [cpubench] | cpu_metric | 4294   | 9532       | 125           |

``` cpubench 15 &; cpubench 15 &; cpubench 15 & ```

| pid  | type       |                 | metric      | start_tick | elapsed_ticks |
|:----:|:----------:|:---------------:|:-----------:|:----------:|:-------------:|
| 8    | [cpubench] | cpu_metric | 1474        | 17735      | 364           |
| 8    | [cpubench] | cpu_metric | 2657        | 18105      | 202           |
| 8    | [cpubench] | cpu_metric | 2618        | 18424      | 205           |
| 8    | [cpubench] | cpu_metric | 1973        | 18742      | 272           |
| 8    | [cpubench] | cpu_metric | 2396        | 19033      | 224           |
| 8    | [cpubench] | cpu_metric | 1656        | 19422      | 324           |
| 8    | [cpubench] | cpu_metric | 4129        | 20065      | 130           |
| 8    | [cpubench] | cpu_metric | 1693        | 20473      | 317           |
| 8    | [cpubench] | cpu_metric | 4036        | 21134      | 133           |
| 8    | [cpubench] | cpu_metric | 1896        | 21377      | 283           |
| 8    | [cpubench] | cpu_metric | 1693        | 21664      | 317           |
| 8    | [cpubench] | cpu_metric | 1903        | 22072      | 282           |
| 8    | [cpubench] | cpu_metric | 2657        | 22678      | 202           |
| 8    | [cpubench] | cpu_metric | 3485        | 23026      | 154           |
| 8    | [cpubench] | cpu_metric | 2618        | 23340      | 205           |
| -----|---------------- | ------------------------ | ----------- | ------------ | ------------|
| 10   | [cpubench] | cpu_metric | 1470        | 17736      | 365           |
| 10   | [cpubench] | cpu_metric | 2810        | 18178      | 191           |
| 10   | [cpubench] | cpu_metric | 1656        | 18636      | 324           |
| 10   | [cpubench] | cpu_metric | 1910        | 19096      | 281           |
| 10   | [cpubench] | cpu_metric | 1693        | 19423      | 317           |
| 10   | [cpubench] | cpu_metric | 2657        | 19803      | 202           |
| 10   | [cpubench] | cpu_metric | 4129        | 20201      | 130           |
| 10   | [cpubench] | cpu_metric | 1726        | 20474      | 311           |
| 10   | [cpubench] | cpu_metric | 2605        | 20861      | 206           |
| 10   | [cpubench] | cpu_metric | 1588        | 21271      | 338           |
| 10   | [cpubench] | cpu_metric | 2064        | 21793      | 260           |
| 10   | [cpubench] | cpu_metric | 1973        | 22073      | 272           |
| 10   | [cpubench] | cpu_metric | 4006        | 22473      | 134           |
| 10   | [cpubench] | cpu_metric | 3555        | 23135      | 151           |
| 10   | [cpubench] | cpu_metric | 4161        | 23552      | 129           |
| -----|---------------- | ------------------------ | ----------- | ------------ | ------------|
| 11   | [cpubench] | cpu_metric | 1474        | 17756      | 364           |
| 11   | [cpubench] | cpu_metric | 2556        | 18373      | 210           |
| 11   | [cpubench] | cpu_metric | 1677        | 18637      | 320           |
| 11   | [cpubench] | cpu_metric | 2209        | 19169      | 243           |
| 11   | [cpubench] | cpu_metric | 1988        | 19528      | 270           |
| 11   | [cpubench] | cpu_metric | 2711        | 19860      | 198           |
| 11   | [cpubench] | cpu_metric | 4129        | 20337      | 130           |
| 11   | [cpubench] | cpu_metric | 2056        | 20591      | 261           |
| 11   | [cpubench] | cpu_metric | 2644        | 20924      | 203           |
| 11   | [cpubench] | cpu_metric | 1616        | 21272      | 332           |
| 11   | [cpubench] | cpu_metric | 1651        | 21663      | 325           |
| 11   | [cpubench] | cpu_metric | 3195        | 22298      | 168           |
| 11   | [cpubench] | cpu_metric | 2556        | 22609      | 210           |
| 11   | [cpubench] | cpu_metric | 4097        | 22889      | 131           |
| 11   | [cpubench] | cpu_metric | 2556        | 23289      | 210           |

``` cpubench 15 &; iobench 15 &; iobench 15 &; iobench 15 & ```

| pid |    type    |            | metric | start_tick | elapsed_ticks |
| :-: | :--------: | :--------: | :----: | :--------: | :-----------: |
|  5  | [cpubench] | cpu_metric |  3947  |    260     |      136      |
|  5  | [cpubench] | cpu_metric |  3890  |    397     |      138      |
|  5  | [cpubench] | cpu_metric |  3976  |    536     |      135      |
|  5  | [cpubench] | cpu_metric |  3976  |    671     |      135      |
|  5  | [cpubench] | cpu_metric |  3947  |    806     |      136      |
|  5  | [cpubench] | cpu_metric |  4066  |    942     |      132      |
|  5  | [cpubench] | cpu_metric |  4066  |    1074    |      132      |
|  5  | [cpubench] | cpu_metric |  4066  |    1207    |      132      |
|  5  | [cpubench] | cpu_metric |  3754  |    1339    |      143      |
|  5  | [cpubench] | cpu_metric |  4066  |    1483    |      132      |
|  5  | [cpubench] | cpu_metric |  4066  |    1616    |      132      |
|  5  | [cpubench] | cpu_metric |  3754  |    1749    |      143      |
|  5  | [cpubench] | cpu_metric |  4036  |    1892    |      133      |
|  5  | [cpubench] | cpu_metric |  3976  |    2026    |      135      |
|  5  | [cpubench] | cpu_metric |  3918  |    2161    |      137      |
|  7  | [iobench]  | io_metric  |   73   |    266     |     1393      |
|  7  | [iobench]  | io_metric  |  176   |    1776    |      580      |
|  7  | [iobench]  | io_metric  |  636   |    2358    |      161      |
|  7  | [iobench]  | io_metric  |  812   |    2520    |      126      |
|  7  | [iobench]  | io_metric  |  846   |    2647    |      121      |
|  7  | [iobench]  | io_metric  |  1013  |    2769    |      101      |
|  7  | [iobench]  | io_metric  |  752   |    2872    |      136      |
|  7  | [iobench]  | io_metric  |  939   |    3015    |      109      |
|  7  | [iobench]  | io_metric  |  673   |    3140    |      152      |
|  7  | [iobench]  | io_metric  |  489   |    3295    |      209      |
|  7  | [iobench]  | io_metric  |  846   |    3506    |      121      |
|  7  | [iobench]  | io_metric  |  758   |    3631    |      135      |
|  7  | [iobench]  | io_metric  |  914   |    3769    |      112      |
|  7  | [iobench]  | io_metric  |  832   |    3882    |      123      |
|  7  | [iobench]  | io_metric  |  581   |    4034    |      176      |
  
| 9   | [iobench] | io_metric | 57   | 267  | 1776 |
| --- | --------- | --------- | ---- | ---- | ---- |
| 9   | [iobench] | io_metric | 274  | 2048 | 373  |
| 9   | [iobench] | io_metric | 758  | 2425 | 135  |
| 9   | [iobench] | io_metric | 652  | 2562 | 157  |
| 9   | [iobench] | io_metric | 605  | 2747 | 169  |
| 9   | [iobench] | io_metric | 860  | 2918 | 119  |
| 9   | [iobench] | io_metric | 1055 | 3043 | 97   |
| 9   | [iobench] | io_metric | 948  | 3142 | 108  |
| 9   | [iobench] | io_metric | 664  | 3262 | 154  |
| 9   | [iobench] | io_metric | 1177 | 3417 | 87   |
| 9   | [iobench] | io_metric | 890  | 3513 | 115  |
| 9   | [iobench] | io_metric | 752  | 3630 | 136  |
| 9   | [iobench] | io_metric | 731  | 3787 | 140  |
| 9   | [iobench] | io_metric | 664  | 3928 | 154  |
| 9   | [iobench] | io_metric | 682  | 4083 | 150  |
| 10  | [iobench] | io_metric | 57   | 574  | 1782 |
| 10  | [iobench] | io_metric | 1204 | 2358 | 85   |
| 10  | [iobench] | io_metric | 731  | 2447 | 140  |
| 10  | [iobench] | io_metric | 781  | 2588 | 131  |
| 10  | [iobench] | io_metric | 605  | 2722 | 169  |
| 10  | [iobench] | io_metric | 522  | 2893 | 196  |
| 10  | [iobench] | io_metric | 527  | 3098 | 194  |
| 10  | [iobench] | io_metric | 832  | 3293 | 123  |
| 10  | [iobench] | io_metric | 696  | 3425 | 147  |
| 10  | [iobench] | io_metric | 764  | 3575 | 134  |
| 10  | [iobench] | io_metric | 800  | 3711 | 128  |
| 10  | [iobench] | io_metric | 731  | 3840 | 140  |
| 10  | [iobench] | io_metric | 839  | 3982 | 122  |
| 10  | [iobench] | io_metric | 1077 | 4117 | 95   |
| 10  | [iobench] | io_metric | 648  | 4233 | 158  |

##### Casos:

* __cpubench__: En el caso en que se ejecutan en paralelo varios procesos 
cpubench, la media de la metrica es considerablemente menor (esto significa una
menor cantidad de operaciones por unidad de tiempo) a el caso en el que se 
ejecuta unicamente un cpubench, esto se explica debido a que, al haber mas de un
cpubench en ejecucion, el procesador hace una mayor cantidad de contextswitch, 
aumentando asi el tiempo de ejecucion de cada proceso.

* __iobench__: En el caso en que se ejecuta un unico cpubench junto con varios
iobench, la metrica no varia mucho con respecto a la ejecucion de un unico 
cpubench en solitario, esto se debe a lo explicado anteriormente en el punto 2
(cpubench termina acaparando el procesador y se ejecuta en su totalidad antes 
que los iobench).

<br>
<br>
5. ¿Es adecuado comparar la cantidad de operaciones de cpu con la cantidad de 
operaciones de iobound?

La conveniencia de comparar operaciones CPU-bound e I/O-bound depende del tipo 
de análisis que el investigador esté realizando, pero en general no suele ser 
adecuado comparar estas operaciones directamente, ya que tienen una naturaleza 
fundamentalmente distinta.

* __Naturaleza de las Operaciones__: Las operaciones _CPU-bound_ están 
completamente ligadas al uso intensivo de la _CPU_, involucrando principalmente 
cálculos y procesamiento. En cambio, las operaciones _I/O-bound_ dependen de la 
interacción con algún dispositivo de entrada o salida y están limitadas por la 
_velocidad y latencia_ de ese subsistema de _I/O_.

* __Interpretación del Resultado__: Comparar directamente la cantidad de 
operaciones puede no proporcionar una visión completa del rendimiento del 
sistema. En su lugar, sería más prudente analizar el __tiempo invertido en cada 
tipo de operación__, ya que esto permite entender mejor cómo se distribuyen las
cargas en el sistema y facilita optimizaciones en términos de planificación y
uso de recursos.

<br>

---

## Experimento 2

### Respuestas

1. ¿Fue necesario modificar las métricas para que los resultados fueran
comparables? ¿Por qué?

Al tener como métrica el cociente entre cantidad de operaciones y tiempo 
transcurrido (ambas tomadas sobre cada iteracion del proceso en cuestion), 
tuvimos que multiplicar la cantidad de operaciones en la métrica para que esta
no se aproxime demasiado al 0, dado que al disminuir el _quantum_ los procesos
hacen más _context switches_ en un lapso más corto, lo que lleva a que el tiempo
de ejecución de los programas sea mayor. Esto implica que la métrica está
dividiendo la misma cantidad de operaciones del experimento anterior por un 
tiempo transcurrido mucho más grande, haciendo que el resultado se aproxime a 
0.

En particular se usaron las siguientes métricas para los _quantums_:

* Para el _quantum_ 10000 se usó:
    
    * cpubench: ```(total_cpu_kops * 10) / elapsed_ticks```
    
    * iobench: ```(total_iops * 1000) / elapsed_ticks```


* Para el _quantum_ 1000 se usó:
    
    * cpubench: ```(total_cpu_kops * 1000) / elapsed_ticks```
    
    * iobench: ```(total_iops * 10000) / elapsed_ticks```


<br>

2. ¿Qué cambios se observan con respecto al experimento anterior? ¿Qué
comportamientos se mantienen iguales?

Reducir el quantum de tiempo tiene un impacto notable en la eficiencia general
del sistema, especialmente en la frecuencia de cambios de contexto y en el
rendimiento de procesos CPU-bound e I/O-bound. 
A su vez los procesos IO-bound y los CPU-bound se ven afectados de forma 
negativa, ya que se puede observar mayor tiempo de respuesta para ambos. 
Se puede ver también que se mantiene igual el comportamiento de espera de los 
procesos IO-bound si se ejecutan en paralelo con los CPU-bound, donde se 
inician al mismo tiempo pero una vez que se bloquean los procesos IO-bound, 
terminan antes los procesos CPU-bound ya que los IO-bound se encuentran en 
estado SLEEPING por mucho tiempo.

![1 iobench y 3 cpubench en paralelo](7_media/Iobench+3CpubenchEscalaLog().png)

<br>

3. ¿Con un quatum más pequeño, se ven beneficiados los procesos iobound o los
procesos cpubound?

Al tener un quantum más pequeño, los procesos CPU-bound se ven afectados en 
su rendimiento por tener una mayor cantidad de interrupciones y ser 
replanificados más frecuentemente. En cambio los procesos IO-bound, pueden 
llegar a ser beneficiados por este mismo motivo, ya que son procesos que 
ceden el CPU reiteradamente y podrían tener acceso de nuevo a este más 
rápidamente.
Hay que tener en cuenta que un quantum demasiado pequeño puede perjudicar a 
todos los procesos, ya que hay un aumento del tiempo en que el SO realiza 
los cambios de contexto.
Tambien podemos notar en la siguiente tabla del experimento:  
```iobench 15 &; cpubench 15 &; cpubench 15 &; cpubench 15 &``` que con un 
_quantum_ de 1000 los _start_ticks_ de los iobench están  encapsulados entre los
_start_ticks_ de los cpubench. Por ejemplo, el último _start_tick_ de los 
iobench es 800912, mientras que podemos ver 1072575 o 1513054  entre los tiempos 
de los cpubench. Es decir, todos los iobench se planifican al menos una vez 
antes de que terminen los cpubench, caso contrario a lo que ocurria con el 
_quanto_ del experimento 1, en el que los cpubench acaparaban el procesador.
A modo de referencia, en el experimento 1, corriendo el mismo comando, sólo las
primeras tres ejecuciones de iobench corrian antes de que terminaran todos los
cpubench (revisar tabla del experimento 1).

| pid |    type    |            | metric | start_tick  | elapsed_ticks |
| :-: | :--------: | :--------: | :----: | :---------: | :-----------: |
|  5  | [iobench]  | io_metric  |  169   |    43452    |     60352     |
|  5  | [iobench]  | io_metric  |  162   |   104121    |     62825     |
|  5  | [iobench]  | io_metric  |  191   |   167128    |     53590     |
|  5  | [iobench]  | io_metric  |  213   |   220928    |     48026     |
|  5  | [iobench]  | io_metric  |  217   |   269110    |     47089     |
|  5  | [iobench]  | io_metric  |  195   |   316355    |     52366     |
|  5  | [iobench]  | io_metric  |  216   |   368901    |     47206     |
|  5  | [iobench]  | io_metric  |  201   |   416299    |     50779     |
|  5  | [iobench]  | io_metric  |  170   |   467269    |     60074     |
|  5  | [iobench]  | io_metric  |  197   |   527624    |     51764     |
|  5  | [iobench]  | io_metric  |  203   |   579544    |     50333     |
|  5  | [iobench]  | io_metric  |  189   |   630066    |     53993     |
|  5  | [iobench]  | io_metric  |  189   |   684305    |     54173     |
|  5  | [iobench]  | io_metric  |  165   |   738637    |     61950     |
|  5  | [iobench]  | io_metric  |  175   | __800912__  |     58204     |
|     |            |            |        |             |               |
|  7  | [cpubench] | cpu_metric |  2751  |    44933    |    195080     |
|  7  | [cpubench] | cpu_metric |  3295  |   243263    |    162895     |
|  7  | [cpubench] | cpu_metric |  2885  |   409928    |    186071     |
|  7  | [cpubench] | cpu_metric |  2834  |   600397    |    189423     |
|  7  | [cpubench] | cpu_metric |  4660  |   793607    |    115195     |
|  7  | [cpubench] | cpu_metric |  8105  |   912159    |     66229     |
|  7  | [cpubench] | cpu_metric |  8689  |   981484    |     61781     |
|  7  | [cpubench] | cpu_metric |  7201  |   1046282   |     74548     |
|  7  | [cpubench] | cpu_metric |  6985  |   1124859   |     76851     |
|  7  | [cpubench] | cpu_metric |  6430  |   1205159   |     83483     |
|  7  | [cpubench] | cpu_metric |  8398  |   1291448   |     63919     |
|  7  | [cpubench] | cpu_metric | 10011  |   1358364   |     53623     |
|  7  | [cpubench] | cpu_metric | 11390  |   1414790   |     47129     |
|  7  | [cpubench] | cpu_metric | 11757  |   1464954   |     45657     |
|  7  | [cpubench] | cpu_metric | 12937  | __1513054__ |     41494     |
|     |            |            |        |             |               |
|  9  | [cpubench] | cpu_metric |  6121  |    45055    |     87691     |
|  9  | [cpubench] | cpu_metric |  6495  |   135815    |     82641     |
|  9  | [cpubench] | cpu_metric |  6455  |   221435    |     83164     |
|  9  | [cpubench] | cpu_metric |  6494  |   306815    |     82659     |
|  9  | [cpubench] | cpu_metric |  6426  |   391963    |     83536     |
|  9  | [cpubench] | cpu_metric |  6101  |   477129    |     87984     |
|  9  | [cpubench] | cpu_metric |  6244  |   567348    |     85966     |
|  9  | [cpubench] | cpu_metric |  6425  |   655344    |     83549     |
|  9  | [cpubench] | cpu_metric |  5757  |   741530    |     93237     |
|  9  | [cpubench] | cpu_metric |  6443  |   836657    |     83316     |
|  9  | [cpubench] | cpu_metric |  7276  |   923851    |     73772     |
|  9  | [cpubench] | cpu_metric |  7881  |   1001079   |     68112     |
|  9  | [cpubench] | cpu_metric |  6958  | __1072575__ |     77147     |
|  9  | [cpubench] | cpu_metric |  6685  |   1153267   |     80303     |
|  9  | [cpubench] | cpu_metric |  7159  |   1237690   |     74981     |
|     |            |            |        |             |               |
| 10  | [cpubench] | cpu_metric |  4294  |    45347    |    125014     |
| 10  | [cpubench] | cpu_metric |  4335  |   173629    |    123835     |
| 10  | [cpubench] | cpu_metric |  4333  |   300362    |    123866     |
| 10  | [cpubench] | cpu_metric |  4135  |   427445    |    129804     |
| 10  | [cpubench] | cpu_metric |  4185  |   559899    |    128265     |
| 10  | [cpubench] | cpu_metric |  4193  |   691600    |    128005     |
| 10  | [cpubench] | cpu_metric |  4581  |   822487    |    117177     |
| 10  | [cpubench] | cpu_metric |  5131  |   943091    |    104606     |
| 10  | [cpubench] | cpu_metric |  4757  |   1051171   |    112849     |
| 10  | [cpubench] | cpu_metric |  4400  |   1167527   |    121980     |
| 10  | [cpubench] | cpu_metric |  5764  |   1292960   |     93124     |
| 10  | [cpubench] | cpu_metric |  6826  |   1388614   |     78634     |
| 10  | [cpubench] | cpu_metric |  7537  |   1469877   |     71222     |
| 10  | [cpubench] | cpu_metric | 10892  |   1543439   |     49285     |
| 10  | [cpubench] | cpu_metric | 13235  |   1595070   |     40560     |

---
# Tercera parte

Esta sección del laboratorio pide que agreguemos dos reglas de MLFQ a nuestro
scheduler xv6. Las reglas son:

* __Regla 3__: Cuando un proceso se inicia, su prioridad será la máxima.

* __Regla 4__: Descender de prioridad cada vez que el proceso pasa todo un
_quantum_ realizando cómputo. Ascender de prioridad cada vez que el proceso se 
bloquea antes de terminar su _quantum_

A su vez se pedía modificar la funcion _procdump_ para que imprimiera la 
prioridad de los procesos.

### Implementación

Lo primero que se hizo para implementar lo pedido fue agregar la prioridad y 
la cantidad de veces que fue elegido un proceso en el struct proc. Se hizo de 
la siguiente manera:
El struct proc se encuentra en kernel/proc.h
```
struct proc {
  struct spinlock lock;

  // set the priority and the times scheduled
  uint64 priority;
  uint64 chosen_count;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```
Donde priority y chosen_count son las adiciones.
Luego en param.h se definio NPRIO 3 que marca los niveles de prioridad para 
los procesos.

El resto de la implementación se hizo toda en kernel/proc.c donde se cambiaron
las funciones allocproc(), scheduler() y procdump(). Veamos los cambios en cada
una.

* __allocproc ()__: Lo que hace esta función es que pasa de un proceso en estado
UNUSED a USED y lo inicializa. El cambio que se realizo fue añadir dos lineas
luego de que se cambia el estado a USED, estas lineas setean la prioridad a 
NPRIO-1 y el chosen_count a 0.

```
found:
  p->pid = allocpid();
  p->state = USED;
  p->priority = NPRIO - 1; //priority set to max
  p->chosen_count = 0;     //sched times set to 0
```

* __procdump()__: La función lo que hace es imprimir por stdout el pid, estado y
nombre de los procesos ejecutandose. La única modificación que se le hizo fue 
que imprimiera tambien la prioridad y las veces seleccionado el proceso.

```
printf("%d %s %s prio: %d chosen: %d", p->pid, state, p->name, p->priority, p->chosen_count);
```

* __scheduler__: El scheduler decide qué procesos se van a ejecutar. Los cambios
que hicimos aquí no afectan su funcionamiento (eso es para la parte 4) si no 
que cambian los valores de _priority_ y _chosen_count_. El cambio fue el 
siguiente: 
Luego de que un proceso fuera seleccionado y pasado de RUNNABLE a RUNNING, el 
_chosen_count_ aumenta en 1. Posteriormente se ejecuta el cambio de contexto, 
entonces se chequea si el estado cambió a SLEEPING o a RUNNABLE. En caso de que
pase lo primero y de que la prioridad no sea ya la máxima, se aumenta. En caso
de que pase lo segundo, se chequea que la prioridad no sea la mínima y se 
decrementa en 1. 

```
    if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        p->chosen_count++; //updates the counter
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        //priority adjusted according to the behaviour
        if (p->state == SLEEPING) {
          if (p->priority < NPRIO - 1) {
            p->priority ++; //Boost the priority if bloqued before quantums ends
          }

        } else if (p->state == RUNNABLE) {
            if (p->priority > 0) {
              p->priority--; //Decrements priority if quantum was used
            }
        }
```

---
# Cuarta parte

* __Casos muchos CPU-Bench__: Con el planificador mlfq cuando hay varios procesos 
que usan intensivamente el cpu, se aprecia un impacto negativo en el rendimiento 
de este tipo de procesos. Esto ocurre porque siempre tienen una baja prioridad por 
lo que se intercalan con los demás procesos. Estos efectos se ven mitigados al 
aumentar el tamaño del quantum.

* __Casos muchos IO-Bench__: Los procesos io-bench tienden a recibir la máxima 
prioridad, si bien esta se mantiene, al haber muchos de estos procesos compiten 
con los otros en la misma prioridad, lo que se ve reflejado en un posible decaimiento 
del rendimiento.

* __Casos CPU-Bench/IO-Bench__: Los procesos de tipo IO-bench y CPU-bench se comportan 
de forma más intercalada en cuanto al acceso al CPU. Con el planificador MLFQ 
intentamos maximizar la responsividad al priorizar los procesos IO-bench.
Los procesos IO-bench deberían experimentar un mejor rendimiento en MLFQ que en el 
planificador original. Esto se debe a que mantienen su prioridad alta y recuperan 
rápidamente el acceso al CPU tras cada operación IO. No lo observamos tanto en 
nuestras mediciones debido a nuestra implementación. Creemos que se produce un 
overhead que reduce la eficiencia, ya que significa que una parte de los recursos 
del sistema se dedica a tareas administrativas en lugar de a ejecutar los procesos 
correspondientes.

* __¿Se puede producir starvation en el nuevo planificador?__

Sí, el problema de starvation puede producirse en el planificador MLFQ. Los procesos 
que usan más CPU son movidos progresivamente a colas de menor prioridad, mientras que 
los procesos io-bench que realizan ráfagas cortas de CPU  tienden a mantenerse en 
colas de mayor prioridad, donde se les otorga un mayor acceso al CPU, lo que no permiten 
el acceso a los procesos CPU-bench. Por la falta de una regla de promoción, en nuestra 
implementación del planificador, los procesos CPU-bench mantienen su baja prioridad y 
no son elegidos por el planificador.