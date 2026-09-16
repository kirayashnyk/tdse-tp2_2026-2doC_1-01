El módulo implementa la lógica de control del sistema (`task_system`) mediante una máquina de estados finitos (FSM) no bloqueante orientada a eventos. Procesa eventos recibidos en la cola FIFO `event_task_system_queue` y los traduce en comandos de salida hacia la interfaz del actuador `put_event_task_actuator`.

**Comportamiento de `task_system_normal_statechart**`

La función gestiona el estado lógico del sistema en la variable global/lista para el índice `NORMAL`:

1. **Extracción de eventos:** Llama a `any_event_task_system()`. Si hay eventos pendientes en la cola, ejecuta `get_event_task_system()`, almacena el evento en `task_system_dta_list[NORMAL].event` y establece `flag = true`.


2. **Evaluación de la máquina de estados:**
* **`ST_SYS_IDLE`:** Si `flag == true` y el evento es `EV_SYS_ACTIVE`, limpia la bandera (`flag = false`), llama a `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)` y cambia al estado `ST_SYS_ACTIVE`.


* **`ST_SYS_ACTIVE`:** Si `flag == true` y el evento es `EV_SYS_IDLE`, limpia la bandera (`flag = false`), llama a `put_event_task_actuator(EV_LED_IDLE, ID_LED_A)` y regresa a `ST_SYS_IDLE`.


* **`default` (recuperación de errores):** Restablece `tick = DEL_SYS_MIN` (0), `state = ST_SYS_IDLE`, `event = EV_SYS_IDLE` y `flag = false`.





---

**Evolución de variables en `task_system_dta_list**`

La unidad de medida de `tick` son **milisegundos (mS)**. El índice utilizado es `index = 0` (`NORMAL`).

| Etapa de Ejecución | `index` | `tick` [mS] | `state` | `event` | `flag` |
| --- | --- | --- | --- | --- | --- |
| **Inicio (`task_system_init`)**<br> | 0 | 0 | `ST_SYS_IDLE` (0) | `EV_SYS_IDLE` (0) | `false`<br> |
| **`task_system_update` (sin eventos)**<br> | 0 | 0 | `ST_SYS_IDLE` (0) | `EV_SYS_IDLE` (0) | `false`<br> |
| **`task_system_update` (recibe `EV_SYS_ACTIVE`)**<br> | 0 | 0 | Transiciona a `ST_SYS_ACTIVE` (1) | `EV_SYS_ACTIVE` (1) | `true` $\rightarrow$ `false`<br> |
| **`task_system_update` (recibe `EV_SYS_IDLE`)**<br> | 0 | 0 | Transiciona a `ST_SYS_IDLE` (0) | `EV_SYS_IDLE` (0) | `true` $\rightarrow$ `false`<br> |

---

**Evolución de variables en `event_task_system_queue**`

La cola almacena hasta 16 elementos (`QUEUE_LENGTH`), indexados de `i = 0` a `15`.

| Etapa de Ejecución | `head` | `tail` | `count` | `queue[i]` (Contenido) |
| --- | --- | --- | --- | --- |
| **Inicio (`init_event_task_system`)**<br> | 0 | 0 | 0 | `queue[0..15]` = `255` (`EMPTY`)|
| **Llega evento `EV_SYS_ACTIVE**`<br> | 1 | 0 | 1 | `queue[0]` = `1` (`EV_SYS_ACTIVE`)|
| **Procesado en `task_system_update**`<br> | 1 | 1 | 0 | `queue[0]` = `255` (`EMPTY`)|
| **Llega evento `EV_SYS_IDLE**`<br> | 2 | 1 | 1 | `queue[1]` = `0` (`EV_SYS_IDLE`)|

---

**Evolución de variables en `task_actuator_dta_list**`

El actuador configurado es el LED principal con `identifier = 0` (`ID_LED_A`).

| Etapa de Ejecución | `identifier` | `task_actuator_dta_list[0].event` | `task_actuator_dta_list[0].flag` |
| --- | --- | --- | --- |
| **Inicio**<br> | 0 (`ID_LED_A`)| Valor por defecto de inicialización | `false`<br> |
| **Transición a activo en `task_system_update**`<br> | 0 (`ID_LED_A`) | `EV_LED_ACTIVE` (1) | `true`<br> |
| **Transición a inactivo en `task_system_update**`<br> | 0 (`ID_LED_A`) | `EV_LED_IDLE` (0) | `true`<br> |
