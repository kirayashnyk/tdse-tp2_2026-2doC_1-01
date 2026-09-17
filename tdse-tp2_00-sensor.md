El código analizado implementa la gestión de un botón físico (`ID_BTN_A`) mediante una máquina de estados finitos (FSM) no bloqueante. Detecta transiciones de pulsación y liberación, enviando eventos a una cola circular de mensajes (`event_task_system_queue`) que conecta con la tarea principal del sistema (`task_system`).

**Comportamiento de `task_sensor_statechart(uint32_t index)`**

Esta función implementa la lógica de control del sensor seleccionado por `index`:

1. **Lectura de entrada:** Lee el pin GPIO (`HAL_GPIO_ReadPin`). Si coincide con el estado configurado como `pressed`, asigna `event = EV_BTN_DOWN`; de lo contrario, asigna `event = EV_BTN_UP`.


2. **Evaluación de transiciones de estado:**
* **`ST_BTN_IDLE`:** Si el evento es `EV_BTN_DOWN`, llama a `put_event_task_system(EV_SYS_ACTIVE)` para notificar al sistema y cambia el estado a `ST_BTN_ACTIVE`.


* **`ST_BTN_ACTIVE`:** Si el evento cambia a `EV_BTN_UP`, llama a `put_event_task_system(EV_SYS_IDLE)` y regresa el estado a `ST_BTN_IDLE`.


* **`default` (recuperación de errores):** Restablece `tick = DEL_BTN_MIN` (0), `state = ST_BTN_IDLE` y `event = EV_BTN_UP`.





---

**Evolución de las variables de `task_sensor_dta_list`**

La unidad de medida de `tick` son **milisegundos (mS)**. Dado que existe un único sensor definido en `task_sensor_cfg_list` (`SENSOR_CFG_QTY = 1`), la variable `index` vale siempre **0**.

| Etapa del ciclo de vida | `index` | `tick` [mS] | `state` | `event` |
| --- | --- | --- | --- | --- |
| **Inicio (`task_sensor_init`)**<br> | 0 | 0 | `ST_BTN_IDLE` (0) | `EV_BTN_UP` (0)|
| **`task_sensor_update` (sin pulsar)**<br> | 0 | 0 | `ST_BTN_IDLE` (0) | `EV_BTN_UP` (0)|
| **`task_sensor_update` (evento presionar)**<br> | 0 | 0 | Transiciona a `ST_BTN_ACTIVE` (1)| `EV_BTN_DOWN` (1)|
| **`task_sensor_update` (mantiene presionado)**<br> | 0 | 0 | `ST_BTN_ACTIVE` (1)| `EV_BTN_DOWN` (1)|
| **`task_sensor_update` (evento liberar)**<br> | 0 | 0 | Transiciona a `ST_BTN_IDLE` (0)| `EV_BTN_UP` (0)|

---

**Evolución de las variables de la cola `event_task_system_queue`**

La cola implementa un búfer circular con capacidad para 16 elementos (`QUEUE_LENGTH`).

| Etapa de ejecución | `head` | `tail` | `count` | Valor almacenado en `queue[i]` |
| --- | --- | --- | --- | --- |
| **Inicialización (`init_event_task_system`)**<br> | 0 | 0 | 0 | `queue[0..15]` = `255` (`EMPTY`)|
| **Transición a presionado (`EV_BTN_DOWN`)**<br> | 1 | 0 | 1 | `queue[0]` = `EV_SYS_ACTIVE` (1)|
| **Mientras permanece presionado**<br> | 1 | 0 | 1 | Sin cambios|
| **Transición a liberado (`EV_BTN_UP`)**<br> | 2 | 0 | 2 | `queue[1]` = `EV_SYS_IDLE` (0)|

 

(Nota: Los valores de `count` y `tail` asumen que ninguna otra tarea ha consumido los eventos de la cola mediante `get_event_task_system()` durante la secuencia).
