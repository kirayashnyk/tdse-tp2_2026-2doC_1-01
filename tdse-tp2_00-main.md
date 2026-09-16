### Análisis del Código Fuente y Flujo de Ejecución

El código proporcionado corresponde a una aplicación para microcontroladores **STM32F103xB** basada en la librería STM32Cube HAL.

1. **`startup_stm32f103rbtx.s`**: Es el archivo de arranque en ensamblador.


* Define la **Tabla de Vectores de Interrupción** (`g_pfnVectors`), reservando las primeras posiciones para la dirección inicial del puntero de pila (`_estack`) y la dirección de `Reset_Handler`.


* Ejecuta la rutina `Reset_Handler` al energizar o reiniciar el microcontrolador. Esta rutina:


* Llama a `SystemInit()` (que configura los relojes por defecto del sistema).


* Copia la sección `.data` cargada en Flash hacia la memoria RAM (`_sidata` $\to$ `_sdata`).


* Borra y pone a cero la sección `.bss` en RAM (`_sbss` a `_ebss`).


* Llama a inicializadores de C/C++ (`__libc_init_array`).


* Salta a la función `main()`.






2. **`main.c`**: Es el punto de entrada principal del programa.


* Inicializa la interfaz de *Semihosting* si la constante `LOGGER_CONFIG_USE_SEMIHOSTING` está habilitada.


* Ejecuta `HAL_Init()` para restablecer periféricos e iniciar SysTick (por defecto a 1 ms).


* Llama a `SystemClock_Config()`, reconfigurando el reloj principal usando el oscilador interno (HSI) a 8 MHz, dividido por 2 (4 MHz) y multiplicado mediante el PLL ($\times 16$), alcanzando una frecuencia de sistema de **64 MHz**.


* Llama a `MX_GPIO_Init()` y `MX_USART2_UART_Init()` para configurar pines GPIO (LED LD2 en salida, Botón B1 en interrupción EXTI15_10) y el módulo UART2 a 115200 baudios.


* Ejecuta las funciones `app_init()` y el bucle infinito `while(1)` con `app_update()`.




3. **`stm32f1xx_it.c`**: Contiene las Rutinas de Servicio de Interrupción (ISR).


* Define los manejadores para excepciones de Cortex-M3 (`HardFault_Handler`, `MemManage_Handler`, etc.).


* Implementa `SysTick_Handler()`, el cual llama a `HAL_IncTick()` para incrementar el contador global de tiempo del HAL.


* Implementa `EXTI15_10_IRQHandler()`, derivando la atención de la interrupción del botón hacia la librería HAL (`HAL_GPIO_EXTI_IRQHandler(B1_Pin)`).





---

### Evolución de `SystemCoreClock` y `SysTick`

A continuación se detalla la evolución de la variable global de frecuencia `SystemCoreClock` y el temporizador/contador periférico `SysTick` desde el `Reset_Handler` hasta llegar al bucle `while (1)`.

#### 1. Frecuencia de Reloj (`SystemCoreClock`)

* **En `Reset_Handler` $\to$ `SystemInit()**`:
* Al encender el microcontrolador, arranca con el oscilador HSI interno por defecto (8 MHz) sin PLL.
* La función `SystemInit()` fija el valor inicial de la variable en **$8\text{ MHz}$** ($8\,000\,000\text{ Hz}$).


* **En `main()` $\to$ Durante `HAL_Init()**`:
* Mantiene el valor base inicial de **$8\text{ MHz}$**.




* **En `main()` $\to$ Durante `SystemClock_Config()**`:
* Se selecciona la fuente **HSI** ($8\text{ MHz}$) con divisor por $2$ $\to$ $4\text{ MHz}$.


* Se activa el **PLL** con multiplicador $\times 16$:

$$f_{\text{PLL}} = 4\text{ MHz} \times 16 = 64\text{ MHz}$$



* Se conmuta la fuente de reloj del sistema (SYSCLK) a `RCC_SYSCLKSOURCE_PLLCLK`.


* Al finalizar la reconfiguración y recalcular el valor de reloj en la HAL, la variable `SystemCoreClock` finaliza actualizada a **$64\text{ MHz}$** ($64\,000\,000\text{ Hz}$).




* **En el Bucle `while (1)**`:
* La variable `SystemCoreClock` se mantiene estable en **$64\text{ MHz}$**.





#### 2. Periférico / Registro de Recarga (`SysTick`)

* **En `Reset_Handler**`:
* El periférico `SysTick` se encuentra en estado por defecto (deshabilitado, registro `LOAD` en $0$, interrupción deshabilitada).


* **En `main()` $\to$ Durante `HAL_Init()**`:
* `HAL_Init()` invoca `HAL_InitTick()`, el cual configura el periférico `SysTick` para generar una interrupción cada $1\text{ ms}$ basada en el reloj actual ($8\text{ MHz}$).


* El valor de recarga `SysTick->LOAD` se establece en:

$$\text{LOAD} = \frac{8\,000\,000\text{ Hz}}{1000\text{ Hz}} - 1 = 7999$$


* Se habilita el temporizador SysTick, el conteo y su interrupción.


* **En `main()` $\to$ Durante `SystemClock_Config()**`:
* Al reconfigurar el reloj principal a $64\text{ MHz}$, la función `HAL_RCC_ClockConfig()` vuelve a invocar internamente a `HAL_InitTick()` para recalcular la velocidad de parada del temporizador.


* Para mantener el tick de tiempo a $1\text{ ms}$ sobre un reloj de $64\text{ MHz}$, el nuevo valor de recarga se actualiza a:

$$\text{LOAD} = \frac{64\,000\,000\text{ Hz}}{1000\text{ Hz}} - 1 = 63999$$




* **En el Bucle `while (1)**`:
* El periférico `SysTick` se mantiene decrementando cíclicamente desde $63999$ hasta $0$ a una velocidad de $64\text{ MHz}$, disparando la interrupción `SysTick_Handler()` e incrementando el contador global `uwTick` exactamente cada $1\text{ ms}$.





---

### Resumen de Evolución de Variables / Parámetros

| Etapa del Código | `SystemCoreClock` | Valor de Recarga SysTick (`SysTick->LOAD`) | Estado del SysTick |
| --- | --- | --- | --- |
| **`Reset_Handler`** | $8\text{ MHz}$ | $0$ | Deshabilitado |
| **`HAL_Init()`** | $8\text{ MHz}$<br> | $7999$ (Intervalo de $1\text{ ms}$) | Habilitado e Interrumpiendo

 |
| **`SystemClock_Config()`** | $64\text{ MHz}$<br> | $63999$ (Intervalo de $1\text{ ms}$) | Reconfigurado y Activo

 |
| **`while (1)`** | $64\text{ MHz}$<br> | $63999$ | Incrementa `uwTick` cada $1\text{ ms}$<br> |
