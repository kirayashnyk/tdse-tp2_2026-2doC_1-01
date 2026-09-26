# Actividad 03 - Módulo System (System Statechart)

## Registro de Datos en Depuración (Debug)

A continuación se presentan los valores observados en la estructura `task_system_dta_list[0]` durante la ejecución del programa, según la captura obtenida del depurador (Live Expressions):

| Campo (Atributo) | Valor Medido | Unidad / Descripción |
| :--- | :--- | :--- |
| `tick` | 0 | ms (milisegundos) |
| `state` | ST_SYS_WAIT_FOR_CAR_ARRIEVE | Estado actual de la FSM del Sistema |
| `event` | EV_SYS_BTN_C_UP | Último evento procesado (leído de la cola) |
| `flag` | true | Bandera lógica interna del statechart |

## Descripción del Funcionamiento e Integración

En esta etapa se implementó el Diagrama de Estados del modelo **System** en el archivo `task_system.c`. Este módulo actúa como el controlador central de la aplicación: procesa los eventos generados por el módulo Sensor y, según su estado actual, decide qué señales enviar al módulo Actuator.

Para la lógica de esta integración, los sensores físicos fueron mapeados a los siguientes eventos del sistema:
*   **Pulsador externo en D3:** Asignado para disparar `EV_SYS_CAMERA`.
*   **Pulsador externo en D2:** Asignado para disparar `EV_SYS_BUTTON`.
*   **Botón integrado de la placa (B1 Blue):** Asignado para disparar `EV_SYS_SENSOR_COIL`.

### Evolución y comportamiento de las variables

La función `task_system_init()` inicializa la estructura de datos del sistema. Posteriormente, `task_system_update()` ejecuta el statechart de manera periódica utilizando una estructura `switch-case` para evaluar las transiciones válidas según el estado activo:

*   **`event`:** El módulo System no lee los pines directamente, sino que consume los eventos almacenados en la cola (`event_task_system_queue`) que fueron enviados por los sensores. Al procesar un evento (como el `EV_SYS_BTN_C_UP` capturado en el debug), evalúa las guardas lógicas (`if`) del estado actual para determinar si debe realizar una transición.
*   **`state`:** Refleja la etapa lógica del sistema. En la captura, el sistema se encuentra estacionado en `ST_SYS_WAIT_FOR_CAR_ARRIEVE`, un estado de espera donde permanecerá hasta que el evento correspondiente a la llegada del auto (asociado al botón integrado) dispare la transición a la siguiente etapa operativa.
*   **`tick` (ms):** Actúa como temporizador del estado activo. Se utiliza en estados que requieren esperas (timeouts) o rutinas temporizadas. Se incrementa con cada llamado a `task_system_update()` y suele reiniciarse al cambiar de estado.
*   **`flag`:** Es una variable de control booleana (`true` / `false`) que el statechart utiliza internamente para recordar condiciones previas, bloquear repeticiones de acciones o habilitar caminos alternativos dentro de la máquina de estados.

*   El diagrama de estados define el ciclo de operación de una barrera de acceso. El recorrido completo del sistema se da de la siguiente manera:

1.  **Inicialización:** El sistema arranca y por defecto envía señales al actuador para asegurarse de que la barrera esté cerrada (`EV_LED_ON`, `ID_LED_BARRIER_CLOSE`) y apaga el indicador de barrera abierta (`EV_LED_OFF`, `ID_LED_BARRIER_OPEN`). El estado inicial es **`ST_SYS_WAIT_FOR_CAR_ARRIEVE`**.
2.  **Llegada del vehículo:** El sistema permanece inactivo hasta que se detecta la llegada de un auto mediante el evento `EV_SYS_CAMERA` (asignado al botón D3). Esto provoca la transición al estado **`ST_SYS_WAIT_FOR_BUTTON_PRESSED`**.
3.  **Apertura de la barrera:** El conductor debe presionar el botón de acceso. Al recibir el evento `EV_SYS_BUTTON` (botón D2), ocurren tres cosas:
    *   Se reinicia el temporizador (`tick = DEL_SYS_MAX`).
    *   Se ordena abrir la barrera (`EV_LED_BLINK`, `ID_LED_BARRIER_OPEN`).
    *   Se apaga el led de barrera cerrada (`EV_LED_OFF`, `ID_LED_BARRIER_CLOSE`).
    *   El sistema transita al estado **`ST_SYS_WAIT_FOR_BARRIER_OPENED`**.
4.  **Espera de apertura:** En este estado, el sistema simplemente descuenta el temporizador (`tick--`) mientras la barrera física se abre. Cuando el tiempo llega a cero (`tick == 0`), se confirma la apertura total enviando la señal `EV_LED_ON` al ID `ID_LED_BARRIER_OPEN` y se pasa al estado **`ST_SYS_WAIT_FOR_CAR_LEAVES`**.
5.  **Cruce del vehículo:** El sistema espera a que el vehículo termine de pasar. Esto se confirma mediante el sensor magnético o bobina en el piso, representado por el evento `EV_SYS_SENSOR_COIL` (el botón integrado de la placa). Al detectarlo:
    *   Se vuelve a reiniciar el temporizador (`tick = DEL_SYS_MAX`).
    *   Se apaga el indicador de barrera abierta (`EV_LED_OFF`, `ID_LED_BARRIER_OPEN`).
    *   Se indica que la barrera se está cerrando (`EV_LED_BLINK`, `ID_LED_BARRIER_CLOSE`).
    *   Se transita al estado **`ST_SYS_WAIT_FOR_BARRIER_CLOSED`**.
6.  **Cierre y reinicio:** Similar a la apertura, el sistema espera a que la barrera termine de bajar descontando el tiempo (`tick--`). Una vez que llega a cero (`tick == 0`), se envía la señal de que está completamente cerrada (`EV_LED_ON`, `ID_LED_BARRIER_CLOSE`) y el sistema regresa al estado inicial **`ST_SYS_WAIT_FOR_CAR_ARRIEVE`**, cerrando el ciclo y quedando listo para el siguiente vehículo.
