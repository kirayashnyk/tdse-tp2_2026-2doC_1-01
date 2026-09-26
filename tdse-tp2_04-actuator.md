# Actividad 04 - Módulo Actuator (LED Statechart)

## Descripción del Funcionamiento

En esta etapa se integró el módulo Actuator, responsable de controlar las salidas físicas del sistema (los LEDs). La implementación se basa en un diagrama de estados que consume los eventos despachados por el módulo System. A diferencia del Sistema, que maneja la lógica de negocio, el Actuator traduce los comandos lógicos a señales eléctricas utilizando las funciones de la capa de abstracción de hardware (HAL).

### Análisis del Diagrama de Estados

El comportamiento de cada instancia de actuador (LED) se rige por tres estados principales, operando de la siguiente manera:

1.  **Inicialización (`ST_LED_OFF`):** 
    Al iniciar, el sistema ejecuta una acción de entrada (Entry Action) llamando a `HAL_GPIO_WritePin` para forzar el estado bajo (`LED_OFF`) en el pin correspondiente. El estado de reposo predeterminado es **`ST_LED_OFF`**.

2.  **Encendido Fijo (`ST_LED_ON`):**
    Si estando apagado, o en parpadeo, el actuador recibe el evento `EV_LED_ON`, el sistema llama a `HAL_GPIO_WritePin` para poner el pin en alto (`LED_ON`). El sistema permanece en el estado **`ST_LED_ON`** de forma indefinida hasta recibir una nueva orden.

3.  **Parpadeo (`ST_LED_BLINK`):**
    El comportamiento dinámico se da al recibir el evento `EV_LED_BLINK`. Al entrar a este estado, el temporizador interno se inicializa (`tick = DEL_LED_MAX`) y el LED se enciende (`LED_ON`). 
    Dentro del estado **`ST_LED_BLINK`**, ocurren dos situaciones según el tiempo transcurrido:
    *   **Temporización (`tick > 0`):** En cada ciclo del procesador, el temporizador se descuenta (`tick--`).
    *   **Conmutación (`tick == 0`):** Cuando el temporizador expira, se ejecuta la función `HAL_GPIO_TogglePin`, que invierte el estado actual del pin físico (de encendido a apagado, o viceversa). Luego, el temporizador se reinicia (`tick = DEL_LED_MAX`) para mantener el ciclo continuo de parpadeo.

4.  **Apagado y reinicio:**
    Desde cualquier estado (ya sea encendido fijo o parpadeando), la recepción del evento `EV_LED_OFF` detiene las acciones actuales, manda a apagar el pin físico con `HAL_GPIO_WritePin` y devuelve al actuador al estado base **`ST_LED_OFF`**.


### Integración con el Módulo System

La verificación final del funcionamiento del actuador se realizó integrando físicamente un LED como representación visual de la barrera. El módulo Actuator fue conectado directamente a la salida de eventos del módulo System, subordinando el comportamiento lumínico al estado lógico del control de acceso:

*   **Estado Apagado:** El LED permanece apagado (`ST_LED_OFF`) durante las etapas de reposo del sistema, es decir, cuando se aguarda la llegada de un vehículo al sensor magnético (`ST_SYS_WAIT_FOR_CAR_ARRIEVE`) y mientras se espera la interacción del usuario con el botón de ingreso (`ST_SYS_WAIT_FOR_BUTTON_PRESSED`).
*   **Estado de Parpadeo:** Durante las transiciones mecánicas, cuando el sistema espera que la barrera termine de abrirse (`ST_SYS_WAIT_FOR_BARRIER_OPENED`) o de cerrarse (`ST_SYS_WAIT_FOR_BARRIER_CLOSED`), el módulo System envía el evento `EV_LED_BLINK`. Esto provoca que el actuador ingrese al estado `ST_LED_BLINK`, indicando visualmente que la barrera se encuentra en movimiento.
*   **Estado Encendido:** El LED se enciende de forma fija (`ST_LED_ON`) única y exclusivamente cuando el sistema confirma que la barrera está completamente abierta y es seguro avanzar, correspondiente al estado donde se aguarda que el vehículo libere la zona (`ST_SYS_WAIT_FOR_CAR_LEAVES`).
