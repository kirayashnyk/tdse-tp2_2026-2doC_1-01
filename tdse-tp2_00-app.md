El sistema implementa un esquema de arquitectura **Bare Metal** guiado por eventos e impulsado por tiempo (*Event-Triggered System / Time-Triggered Execution*).

### Análisis del Funcionamiento de los Archivos

* **`app.c`**: Contiene la lógica central de la aplicación.
* **`app_init()`**: Inicializa el *logger*, resetea la variable global `g_app_cnt`, habilita el contador de ciclos DWT (`cycle_counter_init()`), ejecuta las funciones de inicialización de cada tarea de la lista (`task_sensor_init`, `task_system_init`, `task_actuator_init`) e inicializa sus métricas de tiempo (NOE, LET, BCET, WCET). Finalmente llama a `app_it_init()`.


* **`app_update()`**: Es el bucle de ejecución principal. Protege la lectura del contador de ticks `g_app_tick_cnt` desactivando interrupciones (`CPSID i`). Si existen ticks pendientes, ejecuta las tareas del arreglo secuencialmente (índices 0, 1 y 2), midiendo con el DWT el tiempo transcurrido en microsegundos. Actualiza las métricas estadísticas para cada tarea (LET, BCET, WCET, NOE) y suma el tiempo al total del superloop (`g_app_runtime_us`).




* **`app_it.c`**: Maneja la interrupción del temporizador del sistema. `app_it_init()` resetea `g_app_tick_cnt` a 0. `HAL_SYSTICK_Callback()` incrementa de forma atómica `g_app_tick_cnt` cada 1 millasegundo (configuración típica del SysTick).


* **`logger.c` y `logger.h`**: Proporcionan funciones de registro/depuración sobre Semihosting o printf. `LOGGER_INFO()` desactiva interrupciones, formatea y transmite el mensaje bloqueando la ejecución.


* **`systick.c`**: Implementa `systick_delay_us()`, que genera retardos bloqueantes en microsegundos mediante la lectura directa del registro `SysTick->VAL`.


* **`dwt.h`**: Módulo en línea (*inline*) que configura los registros DWT (Data Watchpoint and Trace) del núcleo ARM Cortex para medir ciclos de reloj precisos e informarlos en microsegundos mediante `cycle_counter_get_time_us()`.



---

### Evolución de las Variables

**Unidades de medida:**

* `g_app_tick_cnt`: Unidades de Ticks (adimensional / conteo de interrupciones de 1 ms)


* `g_app_runtime_us`: Microsegundos ($\mu s$)


* `index`: Índice del vector de tareas (adimensional, valores 0, 1 y 2)


* `NOE`: *Number of Executions* (adimensional / número de ejecuciones)


* `LET`: *Last Execution Time* ($\mu s$)


* `BCET`: *Best-Case Execution Time* ($\mu s$)


* `WCET`: *Worst-Case Execution Time* ($\mu s$)



Sea $t_i$ el tiempo en $\mu s$ que consume la ejecución de la tarea $i$ en una iteración particular:

#### 1. Durante la inicialización (`app_init()`)

* **`g_app_tick_cnt`**: Se inicializa en $0$.


* **`g_app_runtime_us`**: No se utiliza / Valor indeterminado o $0$.


* **`index`**: Toma los valores 0, 1 y 2 secuencialmente dentro del bucle.


* **Valores iniciales en `task_dta_list[index]`** para los 3 índices ($index \in \{0, 1, 2\}$):


* `NOE` = $0$

* `LET` = $0\ \mu s$

* `BCET` = $1000\ \mu s$

* `WCET` = $0\ \mu s$




#### 2. Primera ejecución del bucle (`app_update()`) tras el primer Tick de SysTick

Cuando el SysTick genera la interrupción, `g_app_tick_cnt` pasa a $1$.

1. `app_update()` detecta `g_app_tick_cnt > 0`, decrementa `g_app_tick_cnt` a $0$ y activa `b_time_update_required = true`.


2. `g_app_runtime_us` se reinicia a $0$.


3. **Para $index = 0$ (Task Sensor):**
* `NOE`: Incrementa a $1$.


* `LET`: Almacena el tiempo consumido $t_0\ \mu s$.


* `BCET`: Como $t_0 < 1000$, se actualiza a $t_0\ \mu s$.


* `WCET`: Como $t_0 > 0$, se actualiza a $t_0\ \mu s$.


* `g_app_runtime_us`: Pasa a valer $t_0\ \mu s$.




4. **Para $index = 1$ (Task System):**
* `NOE`: Incrementa a $1$.


* `LET`: Almacena el tiempo consumido $t_1\ \mu s$.


* `BCET`: Se actualiza a $t_1\ \mu s$.


* `WCET`: Se actualiza a $t_1\ \mu s$.


* `g_app_runtime_us`: Pasa a valer $t_0 + t_1\ \mu s$.




5. **Para $index = 2$ (Task Actuator):**
* `NOE`: Incrementa a $1$.


* `LET`: Almacena el tiempo consumido $t_2\ \mu s$.


* `BCET`: Se actualiza a $t_2\ \mu s$.


* `WCET`: Se actualiza a $t_2\ \mu s$.


* `g_app_runtime_us`: Pasa a valer final de $t_0 + t_1 + t_2\ \mu s$.





#### 3. Sucesivas ejecuciones del bucle (`app_update()`) en la N-ésima iteración

Para cada tarea $index \in \{0, 1, 2\}$ en la iteración $N$:

* **`g_app_tick_cnt`**: Se decrementa en 1 cada vez que se procesa un tick en el bucle. Si no hubo un tick nuevo, permanece en $0$.


* **`NOE`**: Se incrementa secuencialmente ($N$).


* **`LET`**: Toma el tiempo consumido en esa iteración específica ($t_{index, N}\ \mu s$).


* **`BCET`**: Conserva el valor mínimo histórico: $\min(\text{BCET}_{\text{anterior}}, t_{index, N})\ \mu s$.


* **`WCET`**: Conserva el valor máximo histórico: $\max(\text{WCET}_{\text{anterior}}, t_{index, N})\ \mu s$.


* **`g_app_runtime_us`**: Acumula el tiempo total ejecutado en el loop actual: $\sum_{i=0}^{2} t_{i, N}\ \mu s$.



---

### Impacto de usar `LOGGER_INFO()` en `g_app_runtime_us` y `task_dta_list[index].WCET`

El uso de `LOGGER_INFO()` dentro de una tarea o durante el ciclo de ejecución impacta directamente sobre las métricas temporales del sistema:

1. **Aumento significativo de tiempos por I/O bloqueante**: `LOGGER_INFO()` hace uso de `snprintf()` y `logger_log_print_()` (la cual utiliza `printf()` y `fflush(stdout)` a través de Semihosting). Estas operaciones de formateo e I/O son computacionalmente pesadas y sumamente lentas.


2. **Impacto en `task_dta_list[index].WCET**`: Si una tarea invoca `LOGGER_INFO()`, su tiempo de ejecución medido por el DWT dentro del bloque `cycle_counter` abarcará también la impresión del log. Esto provocará un **incremento drástico en el valor de `WCET**` de esa tarea específica (pudiendo pasar de unos pocos microsegundos a varios milisegundos).


3. **Impacto en `g_app_runtime_us**`: Como `g_app_runtime_us` es la suma acumulada de las variables `LET` de cada tarea en el ciclo (`g_app_runtime_us += task_dta_list[index].LET`), **`g_app_runtime_us` se incrementará en la misma proporción**.


4. **Desactivación de interrupciones**: `LOGGER_INFO()` ejecuta `__asm("CPSID i")` al inicio y `__asm("CPSIE i")` al final. Al mantener las interrupciones deshabilitadas mientras realiza transmisiones lentas por Semihosting, puede provocar que se pierdan o retrasen las interrupciones del SysTick, afectando la precisión del conteo de ticks `g_app_tick_cnt` y el determinismo del sistema de tiempo real.





## Métricas de Ejecución de Tareas (`task_dta_list`)

### Tarea 0: Sensor (index = 0)
* **NOE (Number of Executions):** 91713 (adimensional)
* **LET (Last Execution Time):** 4 μs
* **BCET (Best-Case Execution Time):** 4 μs
* **WCET (Worst-Case Execution Time):** 5 μs

### Tarea 1: System (index = 1)
* **NOE (Number of Executions):** 91713 (adimensional)
* **LET (Last Execution Time):** 3 μs
* **BCET (Best-Case Execution Time):** 3 μs
* **WCET (Worst-Case Execution Time):** 5 μs

### Tarea 2: Actuator (index = 2)
* **NOE (Number of Executions):** 91713 (adimensional)
* **LET (Last Execution Time):** 2 μs
* **BCET (Best-Case Execution Time):** 2 μs
* **WCET (Worst-Case Execution Time):** 4 μs
