2024-11-06 12:35

__Status__: #Avanzada 
__Tags__: #FaMAF #SO #Lab #Computación #Concurrencia #Kernel 
# Lab 2 y Lab 3 SO

##### Volver a [[Sistemas Operativos]]


## Lab 2, Semáforos

Este laboratorio consistió en implementar unos semaphores en el sistema operativo XV6. Se implementó en la versión RISC-V de XV6 
en espacio de kernel y deberá proveer syscalls accesibles desde espacio de usuario.

### Implementación

##### kernel/semaphores.c
Implementación de la syscall semaphores.

``` c
#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "riscv.h"
#include "spinlock.h"
#include "proc.h"
#include "defs.h"

#define MAX_SEM 256
#define SEM_ERROR 0
#define SEM_SUCCESS 1

typedef struct
{
    int value;
    struct spinlock lock;
}   semaphore;

semaphore sem_table[MAX_SEM];

//sem_init for the array. It's executed in main.c on the kernel at start.
void
sem_init(void)
{
    for (uint i = 0; i < MAX_SEM; i++)
    {
        initlock(&sem_table[i].lock, "semaphore_lock");
        sem_table[i].value = -1;
    }
}

int
sem_open(int sem, int value)
{
    if (sem < 0 || sem >= MAX_SEM) return SEM_ERROR;
    int ret = SEM_ERROR;

    acquire(&sem_table[sem].lock);
    if (sem_table[sem].value == -1) //If not open yet, do it.
    {
        sem_table[sem].value = value;
        ret = SEM_SUCCESS;
    }
    release(&sem_table[sem].lock);

    return ret;
}

int
sem_close(int sem)
{
   if (sem < 0 || sem >= MAX_SEM) return SEM_ERROR;

    acquire(&sem_table[sem].lock);
    if (sem_table[sem].value > 0)
    {
        sem_table[sem].value = -1;
    }
    release(&sem_table[sem].lock);

    return SEM_SUCCESS;
}

int
sem_up(int sem)
{
    acquire(&(sem_table[sem].lock));

    if (sem_table[sem].value == -1 ) {
        printf("ERROR: Ilegal increase of closed semaphore.\n");
        release(&(sem_table[sem].lock));
        return SEM_ERROR;
    }

    if (sem_table[sem].value == 0)
        wakeup(&(sem_table[sem]));

    (sem_table[sem].value) += 1;

    release(&(sem_table[sem].lock));

    return SEM_SUCCESS;
}

int
sem_down(int sem)
{
 acquire(&(sem_table[sem].lock));

    if (sem_table[sem].value == -1)
    {
        printf("ERROR: Ilegal decrease of closed semaphore.\n");
        release(&(sem_table[sem].lock));
        return SEM_ERROR;
    }

    while (sem_table[sem].value == 0)
    {
        sleep(&(sem_table[sem]), &(sem_table[sem].lock));
    }

    (sem_table[sem].value) -= 1;

    release(&(sem_table[sem].lock));

    return SEM_SUCCESS;
}
```
##### user/pingpong.c

Implementación del programa de usuario que usa la syscall semaphore, para imprimir en pantalla una secuencia de ```ping``` ```pong```
``` c
#include "../kernel/types.h"
#include "user.h"

#define EXIT_SUCCESS 0
#define EXIT_FAILURE -1
#define SEM_PING 1
#define SEM_PONG 2

int
main(int argc, char *argv[])
{
    if (argc != 2)
    {
        printf("ERROR: Must provide an argument\n");
        exit(EXIT_FAILURE);
    }
    uint pingpong_n = atoi(argv[1]);
    if (pingpong_n < 1)
    {
        printf("ERROR: The argument must be greater or equal than 1.\n");
        exit(EXIT_FAILURE);
    }

    sem_open(SEM_PING, 1);
    sem_open(SEM_PONG, 0);

    int pid;
    pid = fork();
    if (pid < 0)
    {
        printf("ERROR: fork() failed\n");
        exit(EXIT_FAILURE);
    }
    else if (pid == 0)
    {
        for (int i = 0; i < pingpong_n; i++)
        {
            sem_down(SEM_PONG);
            printf("\tpong\n");
            sem_up(SEM_PING);
        }
        exit(EXIT_SUCCESS);
    }
    else
    {

        for (int i = 0; i < pingpong_n; i++)
        {
            sem_down(SEM_PING);
            printf("ping\n");
            sem_up(SEM_PONG);
        }

        wait(0);

        sem_close(SEM_PING);
        sem_close(SEM_PONG);
    }

    exit(EXIT_SUCCESS);
}
```

### Syscalls usadas en XV6

1.  *initlock(struct spinlock\*, char\*)*
initlock es una función de la forma:
``` c
//spinlock.h
// Mutual exclusion lock.
struct spinlock {
  uint locked;       // Is the lock held?

  // For debugging:
  char *name;        // Name of lock.
  struct cpu *cpu;   // The cpu holding the lock.
};

//spinlock.c
void
initlock(struct spinlock *lk, char *name)
{
  lk->name = name;
  lk->locked = 0;
  lk->cpu = 0;
}
```
que recibe un puntero a un struct spinlock y un string (para nombrarlo) para inicializar un lock.

2.  *acquire(struct spinlock\*)*
acquire recibe un puntero a un struct spinlock y trata de obtenerlo, si falla se queda esperando hasta que pueda en busy waiting.
``` c
void
acquire(struct spinlock *lk)
{
  push_off(); // disable interrupts to avoid deadlock.
  if(holding(lk))
    panic("acquire");

  // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
  //   a5 = 1
  //   s1 = &lk->locked
  //   amoswap.w.aq a5, a5, (s1)
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section's memory
  // references happen strictly after the lock is acquired.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Record info about lock acquisition for holding() and debugging.
  lk->cpu = mycpu();
}

```
3.  *release(struct spinlock\*)*
release recibe un puntero a un struct spinlock y lo libera para que otro pueda usarlo.
``` c
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  // Tell the C compiler and the CPU to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other CPUs before the lock is released,
  // and that loads in the critical section occur strictly before
  // the lock is released.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Release the lock, equivalent to lk->locked = 0.
  // This code doesn't use a C assignment, since the C standard
  // implies that an assignment might be implemented with
  // multiple store instructions.
  // On RISC-V, sync_lock_release turns into an atomic swap:
  //   s1 = &lk->locked
  //   amoswap.w zero, zero, (s1)
  __sync_lock_release(&lk->locked);

  pop_off();
}

```

4.  *sleep(void\*, struct spinlock\*)*
Sleep  recibe un puntero vacío (una especie de puntero comodín) y un puntero a un struct spinlock. Lo que hace es 
liberar el lock y enviar el proceso a dormir (probablemente por un llamado a I/O), cuando el proceso retorna entonces
adquiere de nuevo el lock inicial. Esta función se utiliza cuando un proceso trata de acceder a un semáforo pero este
está bloqueado, entonces es mandado a dormir.
``` c
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.

  acquire(&p->lock);  //DOC: sleeplock1
  release(lk);

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  release(&p->lock);
  acquire(lk);
}

```

5.  *wakeup(void\*)*
wakeup recibe un puntero vacío (una suerte de comodín) que apunta  a algún proceso y lo saca del modo SLEEP a 
READY para el scheduler (RUNNABLE para XV6). Esta función se utiliza cuando los semáforos se liberan para poder
despertar a los procesos que fueron enviados a dormir.
``` c
// Wake up all processes sleeping on chan.
// Must be called without any p->lock.
void
wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    if(p != myproc()){
      acquire(&p->lock);
      if(p->state == SLEEPING && p->chan == chan) {
        p->state = RUNNABLE;
      }
      release(&p->lock);
    }
  }
}

```

### El uso de los semáforos
El uso de los semáforos se implemento en el archivo [pingpong.c](#####user/pingpong.c) del user space, y la implementación de los mismo fue hecha en
[semaphores.c](#####kernel/semaphores.c).
Primero expliquemos qué son y cómo funcionan los semáforos:
Los **semaphore** o **semáforos** son mecanismos de sincronización utilizados en programación concurrente y sistemas operativos para controlar el acceso a recursos
compartidos y evitar condiciones de carrera. En esencia, un semáforo es una variable entera que solo puede ser manipulada mediante dos operaciones atómicas principales:

1. **`wait` (o `P` o `down`)**: Disminuye el valor del semáforo. Si el valor es mayor que cero, decrece y permite que el proceso continúe. Si es igual a cero, el proceso se bloquea 
   hasta que el semáforo se incremente por otro proceso.
    
2. **`signal` (o `V` o `up`)**: Aumenta el valor del semáforo. Si hay procesos bloqueados en espera del semáforo, uno de ellos se desbloquea y continúa su ejecución.
    
Existen dos tipos principales de semáforos:

- **Semáforos binarios**: Solo pueden tener los valores 0 o 1, y suelen usarse como locks para garantizar que solo un proceso acceda a un recurso en un momento dado.
    
- **Semáforos contadores**: Permiten valores mayores que 1 y se utilizan para controlar el acceso a un recurso con múltiples instancias disponibles, es decir, pasan tantos
  procesos como el valor indique.

En sistemas operativos, los semáforos son cruciales para la sincronización entre procesos y threads, ayudando a coordinar tareas sin interferir entre ellas.

##### **Implementación en hecha en XV6**
Para la implementación que hicimos en este laboratorio se usaron cinco funciones: 

-  **```void sem_init(void);```**
Esta función se encarga de inicializar ```sem_table[MAX_SEM]``` donde esto es un array de 256 elementos del tipo ```semaphore```, es una operación privilegiada
a la cuál el usuario no tiene acceso, si no que se ejecuta al iniciar el sistema XV6. 
Su ejecución es la siguiente: se corre un ciclo en el que se itera por todos los elementos del array llamando a ```initlock()``` para cada uno de ellos, y seteando
los valores (value) de cada semaphore en -1.
```c
//sem_init for the array. It's executed in main.c on the kernel at start.
void
sem_init(void)
{
    for (uint i = 0; i < MAX_SEM; i++) 
    {
        initlock(&sem_table[i].lock, "semaphore_lock");
        sem_table[i].value = -1;
    }
}
```

-  **```int sem_open(int sem, int value);```**
La función sem_open se encarga de abrir un semaphore para el usuario (es decir lo habilita para su uso). Para hacer esto recibe dos argumentos, primero el índice en
el arreglo de la sem_table y luego el valor al que quiere inicializar este. Se chequea, entonces, que el semaphore que se pidió (sem_table\[sem]) no haya sido ya abierto,
esto lo hace viendo si su "value" es igual a -1, lo cual indicaría que está cerrado, entonces procede a abrirlo seteando ese "value" a el valor que se ingresó. En este caso,
y en las próximas funciones, vamos a notar que parte de su cuerpo está encapsulado por los llamados a acquire y release. Esto son las secciones críticas que no deben
ser interrumpidas por el context switch, haciendolas así atómicas. *En las siguientes explicaciones no voy a referir a esto, pero hay que tenerlo en cuenta.*
```c
int
sem_open(int sem, int value)
{
    if (sem < 0 || sem >= MAX_SEM) return SEM_ERROR;
    int ret = SEM_ERROR;

    acquire(&sem_table[sem].lock);
    if (sem_table[sem].value == -1) //If not open yet, do it.
    {
        sem_table[sem].value = value;
        ret = SEM_SUCCESS;
    }
    release(&sem_table[sem].lock);

    return ret;
}
```

-  **```int sem_close(int sem);```**
sem_close es la función hermana de sem_open, que se encarga de hacer lo contrario. Si recibe un semaphore que estaba abierto lo cierra, si ya estaba cerrado no hace nada.
Para lograr esto recibe el índice de la sem_table y compara su "value" con -1 (en realidad ve si es positivo), si difiere entonces lo setea a -1 y retorna. 

```c
int
sem_close(int sem)
{
   if (sem < 0 || sem >= MAX_SEM) return SEM_ERROR;

    acquire(&sem_table[sem].lock);
    if (sem_table[sem].value > 0) 
    {
        sem_table[sem].value = -1;
    }
    release(&sem_table[sem].lock);

    return SEM_SUCCESS;
}
```

-  **```int sem_up(int sem);```**
Esta función y la próxima son los que hacen que los semaphores funcionen cómo funcionan; sem_up recibe el índice de la sem_table y hace un par de chequeos
sobre el semáforo. Primero ve si está abierto, si no devuelve un error, y luego mira si su "value es 0". En este último caso, si es *true*, trata de despertar al proceso,
llamando a wakeup, que probablemente su hermana envió a dormir. Una vez hecho esos chequeos, aumenta el valor de "value" en 1 y retorna con éxito, habilitando
así a que otro proceso pueda entrar a ejecutarse.
```c
int
sem_up(int sem)
{
    acquire(&(sem_table[sem].lock));

    if (sem_table[sem].value == -1 ) {
        printf("ERROR: Ilegal increase of closed semaphore.\n");
        release(&(sem_table[sem].lock));
        return SEM_ERROR;
    }

    if (sem_table[sem].value == 0)
        wakeup(&(sem_table[sem]));

    (sem_table[sem].value) += 1;

    release(&(sem_table[sem].lock));

    return SEM_SUCCESS;
}
```

- **```int sem_down(int sem);```**
sem_down es la función complementaria a sem_up, recibe el índice de la sem_table para obtener el semáforo que quiere el usuario y hace un par de chequeos.
Primero ve si es un semáforo abierto, si no lo es devuelve error, luego entra en un loop que revisa si el valor de "value" es igual a 0, en caso positivo envía al
proceso a dormir con sleep() hasta que sem_up lo despierte llamando a wakeup(), en el que chequeará de nuevo el loop para ver si puede seguir ejecutando o
tiene que ir a dormir de nuevo. En el caso de que salga del loop, es decir, su "value" es mayor a 0, le restará 1 a este y retornará con éxito.
```c
int
sem_down(int sem)
{
 acquire(&(sem_table[sem].lock));

    if (sem_table[sem].value == -1)
    {
        printf("ERROR: Ilegal decrease of closed semaphore.\n");
        release(&(sem_table[sem].lock));
        return SEM_ERROR;
    }

    while (sem_table[sem].value == 0)
    {
        sleep(&(sem_table[sem]), &(sem_table[sem].lock));
    }

    (sem_table[sem].value) -= 1;

    release(&(sem_table[sem].lock));

    return SEM_SUCCESS;
}
```

##### **Uso para el usuario**
Para que el usuario pueda acceder al uso de los semáforos se tubo que incluir el llamado a estas *syscalls* en los archivos ```syscall.c```, ```syscall.h```,```defs.h```,```sysproc.c```, todas
en kernel space, y en el Makefile y en ```user/user.h``` y ```user/usys.pl```.
Una vez ya fueran implementadas, la consigna del laboratorio nos pedía que hiciéramos un programa ejecutable para el usuario en el que se alternara la impresión de las 
palabras "ping" y "pong", usando semáforos y un (sólo un) fork() (para saber más sobre fork referirse a [[Lab 1 SO]]. El output del programa debería se cómo el siguiente:

```  sh
\$ pingpong 6
ping
	pong
ping
	pong
ping
	pong
ping
	pong
ping
	pong
ping
	pong
\$
```
Donde el código de este puede ser visto más [arriba](#####user/pingpong.c).
-  **Idea detrás del código**:  Primero que nada, el programa debía recibir como input del usuario el número de secuencias de ping pong que deseaba ejecutar. Para 
lograr esto, el programa recibe un argumento en forma de entero. El problema surge de que al pasarlo por terminal este termina siendo un string, por lo tanto aprovechamos
una función ya implementada en XV6 que es `atoi()`, la cuál convierte cualquier string a entero. Su implementación es:

``` c
int
atoi(const char *s)
{
  int n;

  n = 0;
  while('0' <= *s && *s <= '9')
    n = n*10 + *s++ - '0';
  return n;
}

```
Una vez solucionado ese problema, lo siguiente es manejar la lógica del programa. Sabíamos que teniamos que incluir un fork, así que hicimos eso primero y separamos el  *child* del *parent*. En el hijo decidimos que se ejecutaría la parte de *pong* y en el padre la de *ping*, usando simplemente un `printf`.
Ahora había que ver cómo alternabamos las salidas para que cumpliera con lo que se pedía. Claro está que para esto se necesita usar los semáforos que
habíamos implementado. Entonces lo primero fue abrir dos semáforos antes de la llamada del fork y luego poner las llamadas a sem_up y sem_down
donde correspondieran.
¿Por qué dos semáforos? porque necesitamos ir alternando las salidas, es decir, cuando un proceso ejecuta pong se debe quedar esperando hasta que
otro proceso ejecute ping. Para lograr esto un proceso resta su propio "value" de su semáforo correspondiente, mientras que aumenta el de su proceso
"rival".
La implementación fue así:

```c
//abiertos antes del fork, con valores 1 y 0 para que empiece
//uno sólo y no los dos a la vez
    sem_open(SEM_PING, 1);
    sem_open(SEM_PONG, 0);
.................................
................................
//child
	for (int i = 0; i < pingpong_n; i++) 
	{
	//originalmente setado a 0, no ejecuta la primera vez
		sem_down(SEM_PONG);
		printf("\tpong\n");
		sem_up(SEM_PING); //aumenta el de ping
	}
    exit(EXIT_SUCCESS);
.................................
.................................
//parent
	for (int i = 0; i < pingpong_n; i++) 
	{
	//originalmente seteado a 1, sí ejecuta primero
		sem_down(SEM_PING);
		printf("ping\n");
		sem_up(SEM_PONG); //aumenta pong
	}

	wait(0); //espera a que child termine

	//cierra los semáforos
	sem_close(SEM_PING);
	sem_close(SEM_PONG);
```
Luego de que se ejecutan los loops la cantidad de veces necesarias, en este caso pinpong_n veces que es el número que ingresa el usuario,
se espera a que termine el hijo, se cierran los semáforos y se retorna con éxito.
Omití los manejos de error y cómo se dividen el fork dado que no es relevante para el objetivo de este laboratório. En caso de duda se debería 
revisar [[Lab 1 SO]] que debería cubrir esos temas.
## Lab 3, Scheduler

Esto es una especie de resumen para estudiar, si se quiere ver el informe, referirse aquí [[INFORME]]

#### Objetivos del lab
El planificador apropiativo de `xv6-riscv` utiliza un algoritmo sencillo para distribuir tiempo de procesador entre los procesos en ejecución, pero esto
tiene un costo aparejado. Los objetivos de este laboratorio son: 
-  Estudiar el funcionamiento del scheduler original de xv6-riscv 
-  Analizar los procesos que se benefician/perjudican con esta decisión de diseño 
- Desarrollar una implementación reemplazando la política de planificación por una propia que deberá respetar ciertas condiciones 
- Analizar cómo la nueva política afecta a los procesos en comparación con el planificador original.


### Programas bench

Por parte de la cátedra, fuimos provistos por dos programas para testear el scheduler de xv6. Uno dedicado a tareas cpu-intensivas, y otro a
tareas io-intensivas. Dispondré aquí las implementaciones pues serán útiles más tarde.

##### IOBENCH
```c
#define IO_OPSIZE 64
#define IO_EXPERIMENT_LEN 512

static char data[IO_OPSIZE];

int
io_ops()
{
    int rfd, wfd;

    int pid = getpid();

    // Crear un path unico de archivo
    char path[] = "12iops";
    path[0] = '0' + (pid / 10);
    path[1] = '0' + (pid % 10);

    wfd = open(path, O_CREATE | O_WRONLY);

    for(int i = 0; i < IO_EXPERIMENT_LEN; ++i){
      write(wfd, data, IO_OPSIZE);
    }

    close(wfd);

    rfd = open(path, O_RDONLY);

    for(int i = 0; i < IO_EXPERIMENT_LEN; ++i){
      read(rfd, data, IO_OPSIZE);
    }

    close(rfd);
    return 2 * IO_EXPERIMENT_LEN;
}

void
iobench(int N, int pid)
{
  memset(data, 'a', sizeof(data));
  uint64 start_tick, end_tick, elapsed_ticks, metric;
  int total_iops;

  int *measurements = malloc(sizeof(int) * N);

  for (int i = 0; i < N; i++){
    start_tick = uptime();

    // Realizar escrituras y lecturas de archivos
    total_iops = io_ops();

    end_tick = uptime();
    elapsed_ticks = end_tick - start_tick;
    metric = (total_iops * 100)/elapsed_ticks;  // Cambiar esto por la métrica adecuada
    measurements[i] = metric;
    printf("%d\t[iobench]\tio_metric\t%d\t%d\t%d\n",
           pid, metric, start_tick, elapsed_ticks);
  }
}

int
main(int argc, char *argv[])
{
  int N, pid;
  if (argc != 2) {
    printf("Uso: benchmark N\n");
    exit(1);
  }

  N = atoi(argv[1]);  // Número de repeticiones para los benchmarks
  pid = getpid();
  iobench(N, pid);

  exit(0);
}
```

##### CPUBENCH
``` c
#define CPU_MATRIX_SIZE 128
#define CPU_EXPERIMENT_LEN 256

#define MEASURE_PERIOD 1000


// Multiplica dos matrices de tamaño CPU_MATRIX_SIZE x CPU_MATRIX_SIZE
// y devuelve la cantidad de operaciones realizadas / 1000
int
cpu_ops_cycle() {
  int kops_matmul = CPU_MATRIX_SIZE * CPU_MATRIX_SIZE * CPU_MATRIX_SIZE / 1000;
  float A[CPU_MATRIX_SIZE][CPU_MATRIX_SIZE];
  float B[CPU_MATRIX_SIZE][CPU_MATRIX_SIZE];
  float C[CPU_MATRIX_SIZE][CPU_MATRIX_SIZE];

  // Inicializar matrices con valores arbitrarios
  for (int i = 0; i < CPU_MATRIX_SIZE; i++) {
    for (int j = 0; j < CPU_MATRIX_SIZE; j++) {
        A[i][j] = i + j;
        B[i][j] = i - j;
    }
  }

  // Multiplicar matrices N veces
  for (int n = 0; n < CPU_EXPERIMENT_LEN; n++) {
    for (int i = 0; i < CPU_MATRIX_SIZE; i++) {
      for (int j = 0; j < CPU_MATRIX_SIZE; j++) {
        C[i][j] = 0.0f;
        for (int k = 0; k < CPU_MATRIX_SIZE; k++) {
          C[i][j] += 2.0f * A[i][k] * B[k][j];
        }
      }
    }
  }

  return (kops_matmul * CPU_EXPERIMENT_LEN);
}

void
cpubench(int N, int pid) {
  uint64 start_tick, end_tick, elapsed_ticks, total_cpu_kops, metric;
  int *measurements = malloc(sizeof(int) * N);

  // Realizar N ciclos de medicion
  for(int i = 0; i < N; ++i) {
    total_cpu_kops = 0;
    start_tick = uptime();

    total_cpu_kops = cpu_ops_cycle();

    end_tick = uptime();
    elapsed_ticks = end_tick - start_tick;

    // TODO: Cambiar esto por la métrica adecuada
    metric = total_cpu_kops/elapsed_ticks;
    measurements[i] = metric;
    printf("%d\t[cpubench]\tcpu_metric\t%d\t%d\t%d\n",
           pid, metric, start_tick, elapsed_ticks);
  }
}

int
main(int argc, char *argv[])
{
  int N, pid;
  if (argc != 2) {
    printf("Uso: benchmark N\n");
    exit(1);
  }

  N = atoi(argv[1]);  // Número de repeticiones para los benchmarks
  pid = getpid();
  cpubench(N, pid);

  exit(0);
}
```
##### Métrica

En ambos programas hay una variable definida como metric. Esta originalmente era el número de operaciones que se ejecutaban (cpubound para 
cpubench y iobound para iobench), era parte de la consigna cambiarla por algo que pudieramos ir siguiendo para obtener información. Terminamos
cambiandola por cantidad de operaciones dividido tiempo transcurrido (que fue sugerido por los profes en el laboratorio, nos faltó creatividad).
### Scheduler RR 
El scheduler original que trae xv6 es uno de Round Robin que obtiene los procesos de una lista de ellos y chequea que estén en el estado RUNNABLE,
si cumplen con esa condición los pasa a RUNNING y hace el context switch. Este scheduler está en kernel/proc.c y su implementación es:

```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();

  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();

    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
      }
      release(&p->lock);
    }
  }
}
```
### Quantum
Cada proceso que es seleccionado por el scheduler debe ser ejecutado, pero... ¿por cuánto tiempo? ¿hasta que termine?. Bueno, esto ya viene
especificado dentro de los sistemas operativos y se suele llamar quanto, que es una unidad de tiempo en la que los procesos pueden correr,
cuando se acaba, deben ser sacador de RUNNING y llamar de nuevo al scheduler. En particular xv6 tiene un quantum de 1.000.000, que 
emulado por qemu, es aproximadamente $1/10s$. 
Este quanto está implementado en kernel/start.c en la función timerinit() con el nombre interval:

```c
void
timerinit()
{
  // each CPU has a separate source of timer interrupts.
  int id = r_mhartid();

  // ask the CLINT for a timer interrupt.
  int interval = 1000000; // cycles; about 1/10th second in qemu.
  *(uint64*)CLINT_MTIMECMP(id) = *(uint64*)CLINT_MTIME + interval;

  // prepare information in scratch[] for timervec.
  // scratch[0..2] : space for timervec to save registers.
  // scratch[3] : address of CLINT MTIMECMP register.
  // scratch[4] : desired interval (in cycles) between timer interrupts.
  uint64 *scratch = &timer_scratch[id][0];
  scratch[3] = CLINT_MTIMECMP(id);
  scratch[4] = interval;
  w_mscratch((uint64)scratch);

  // set the machine-mode trap handler.
  w_mtvec((uint64)timervec);

  // enable machine-mode interrupts.
  w_mstatus(r_mstatus() | MSTATUS_MIE);

  // enable machine-mode timer interrupts.
  w_mie(r_mie() | MIE_MTIE);
}
```
Para los distintos experimentos que tendremos que correr en este laboratorio, se nos irá pidiendo que achiquemos el quanto y describamos
el comportamiento de los programas.
A modo de resumen simple, si se quieren los detalles ir al [[INFORME]], cando el quanto se achica los procesos cpubound sufren mucho y los
iobound también lo hacen, pero no tanto, pues como llaman a sleep muy seguido y no tienen tantas operaciones, no sufren tanto el context
switch.
Cuando se implemente MLFQ y se midan los mismos experimentos pero en ese scheduler, vamos a ver resultados similares, pero algo
distintos. Con MLFQ los procesos iobound se benefician pues elevan su prioridad cada vez que se bloquean, mientras que los cpubound
pierden prioridad y quedan en un  posible dilema de starvation (ver la siguiente sección para entender MLFQ). Los cambios entre los 
experimentos no son tan notables por el hecho de que los iobound se bloquean tan seguido que no importa mucho su prioridad, por lo que
sigue eligiendo los cpubound muchas veces.
Al achicar el quanto con MLFQ se nota un poco más el cambio en las prioridades. Los cpubound sufren mucho, y los iobound sufren tambien
pero menos, como en el caso RR. Pero aquí podemos notar que la mayoría de procesos llamados de iobound fueron ejecutados antes de que
terminen todos los cpubound, caso que no ocurria antes. Es decir, hay un favoritismo, aunque poco perceptible para los iobound. En casos 
extremos se puede notar el starvation que sufren los procesos de colas bajas.

### Scheduler MLFQ
Luego de haber hecho todos los experimentos con el Round Robin, se nos pide que implementemos un MLFQ para reemplazarlo, con las
siguientes reglas:

1. Si el proceso A tiene mayor prioridad que el proceso B, corre A (no B).
2. Si dos procesos A y B tienen la misma prioridad, corre el que menos veces fue elegido por el planificador.
3. Cuando un proceso se inicia, su prioridad será máxima.
4. Descender de prioridad cada vez que el proceso gasta su quantum realizando cómputo. Ascender de prioridad cada vez que el proceso
se bloquea antes de terminar su quanto.

Lo primero que se hizo fue modificar el struct proc (kernel/proc.h), agregando dos opciones más, priority y chosen_count. La primera llevaba registro
de la cola de prioridad a la que se le había asignado, mientras que chosen_count guardaba las veces que el proceso había sido corrido. Luego de eso,
cambiamos la implementación de procdump para que también mostrara estos dos datos nuevos. La piroridad máxima para este trabajo fue de 3, por 
lo que en kernel/params.h definimos un macro NPRIO 3, que usamos para setear la prioridad máxima a NPRIO-1.
Ahora viene lo importante, hacer el cambio en el mismo scheduler. Creo que será más fácil mostrar primero el código y después tratar de explicar
lo que hicimos.

```c
void
scheduler(void)
{
  //highest priority proc for priority identification
  struct proc *highest_priority_proc;
  struct proc *p;
  struct cpu *c = mycpu();

  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();

    //initialization of highest_priority_proc as NULL.
    highest_priority_proc = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        if (!highest_priority_proc || //first cicle process
            p->priority > highest_priority_proc->priority ||   // Rule 1.
            (p->priority == highest_priority_proc->priority && //Rule 2.
            p->chosen_count < highest_priority_proc->chosen_count)) {

          // Releases the lock of the previous process
          if(highest_priority_proc) {
            release(&highest_priority_proc->lock);
          }
          //sets p as the new highest priority process
          highest_priority_proc = p;

        } else {
          //Releases the lock of the process if not selected.
          release(&p->lock);
        }

      } else {
      // if !RUNNABLE release the lock
      release(&p->lock);
      }
    }
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
      if (highest_priority_proc && highest_priority_proc->state == RUNNABLE) {
        p = highest_priority_proc;
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
      release(&p->lock);
    }
  }
}
```
Antes teníamos un loop en el que cada vez que se topaba con un proceso en estado RUNNABLE, lo preparaba para el cambio de contexto. Ahora,
el loop está encargado de realizar todos los chequeos de prioridad para todos los procesos del arreglo, salvo que encuentre alguno que cumpla las
condiciones rápido. Esto ralentiza el scheduler porque es una manera ineficiente de elegir el próximo proceso, pero funciona. Luego de que realiza
todos los chequeos (la lógica de los ifs y del puntero auxiliar quedan para el lector) realiza las asignaciones habituales que ya hacía el scheduler 
anterior. La única diferencia es que ahora hay que usar el puntero auxiliar para recuperar el proceso, y se deben  hacer una sume de 1 para chosen_count.
Luego de que hace el cambio de contexto se chequean las reglas 3 y 4, si el proceso está en SLEEPING, es decir, bloqueado, se le aumenta la prioridad.
Si el proceso sigue en RUNNABLE se le reduce la prioridad.
El planteamiento es sencillo, aunque no optimo.


## Referencias

-  [[Sistemas Operativos material]]
-  [[Lab2 - Enunciado.pdf]]
-  [[Lab3 - Enunciado.pdf]]
-  [xv6 Manual](https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf)
