**Análisis y Explicación del Código Fuente**

El código implementa una máquina de estados finitos (FSM) no bloqueante para gestionar un actuador (LED) mediante una arquitectura orientada a tareas e interfaces.

* **`task_actuator_attribute.h`**: Define tipos de datos (enums para estados, eventos e identificadores) y las estructuras de configuración (`task_actuator_cfg_t`) y datos genéricos (`task_actuator_dta_t`).


* **`task_actuator.c`**: Contiene la lógica del actuador. `task_actuator_init()` inicializa variables y hardware. `task_actuator_update()` recorre los elementos y ejecuta la FSM mediante `task_actuator_statechart()`.


* **`task_actuator_interface.c`**: Expone la función `put_event_task_actuator()`, la cual actúa como interfaz para recibir eventos externos y notificar a la tarea mediante la bandera `flag`.



---

**Evolución de Variables de la Tarea (`index`, `tick`, `state`, `event`, `flag`)**

Dado que `ACTUATOR_DTA_QTY = 1`, la variable `index` tomará únicamente el valor `0` durante los bucles de ejecución.

* **Unidad de medida de `tick**`: La unidad de medida de `tick` no se incrementa internamente dentro de `task_actuator.c`, pero representa un valor temporal en **milisegundos (mS)**.



| Instancia de Ejecución | `index` | `tick` | `state` | `event` | `flag` |
| --- | --- | --- | --- | --- | --- |
| **Al iniciar (`task_actuator_init`)**<br> | `0` | *Indefinido* | `ST_LED_IDLE`<br> | `EV_LED_IDLE`<br> | `false`<br> |
| **`task_actuator_update` (Sin eventos externos)**<br> | `0` | Sin cambio | `ST_LED_IDLE` | `EV_LED_IDLE` | `false` |
| **Recepción de evento `EV_LED_ACTIVE**` (vía `put_event...`)| `0` | Sin cambio | `ST_LED_IDLE` | `EV_LED_ACTIVE`<br> | `true`<br> |
| **`task_actuator_update` (Siguiente ejecución tras evento)**<br> | `0` | Sin cambio | `ST_LED_ACTIVE`<br> | `EV_LED_ACTIVE` | `false`<br> |
| **Recepción de evento `EV_LED_IDLE**` (vía `put_event...`)| `0` | Sin cambio | `ST_LED_ACTIVE` | `EV_LED_IDLE`<br> | `true`<br> |
| **`task_actuator_update` (Siguiente ejecución tras evento)**<br> | `0` | Sin cambio | `ST_LED_IDLE`<br> | `EV_LED_IDLE` | `false`<br> |

---

**Comportamiento de `task_actuator_statechart(uint32_t index)`**

La función evalúa el estado actual guardado en `task_actuator_dta_list[index]`:

* **En estado `ST_LED_IDLE**`: Si `flag == true` y `event == EV_LED_ACTIVE`, limpia la bandera (`flag = false`), enciende el LED físico a través de HAL GPIO (`led_on`) y transiciona al estado `ST_LED_ACTIVE`.


* **En estado `ST_LED_ACTIVE**`: Si `flag == true` y `event == EV_LED_IDLE`, limpia la bandera (`flag = false`), apaga el LED físico (`led_off`) y transiciona al estado `ST_LED_IDLE`.


* **Estado `default` (Recuperación de fallos)**: Si el estado no es válido, asigna `tick = 0ul` (`DEL_LED_MIN`), e reinicializa el estado a `ST_LED_IDLE`, el evento a `EV_LED_IDLE` y `flag` a `false`.



---

**Evolución de Variables de Interfaz (`identifier`, `event`, `flag`)**

La función de interfaz `put_event_task_actuator(event, identifier)` se llama desde fuera para inyectar eventos enviando el ID del actuador objetivo (`identifier = ID_LED_A = 0`).

| Instancia / Llamada | `identifier` | `event` | `flag` |
| --- | --- | --- | --- |
| **Inicio (`task_actuator_init`)**<br> | `0` | `EV_LED_IDLE`<br> | `false`<br> |
| **Ejecución `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)**`<br> | `0` (`ID_LED_A`)

 | `EV_LED_ACTIVE`<br> | `true`<br> |
| **Tras procesar en `task_actuator_update**`<br> | `0` | `EV_LED_ACTIVE` | `false`<br> |
| **Ejecución `put_event_task_actuator(EV_LED_IDLE, ID_LED_A)**`<br> | `0` (`ID_LED_A`)

 | `EV_LED_IDLE`<br> | `true`<br> |
| **Tras procesar en `task_actuator_update**`<br> | `0` | `EV_LED_IDLE` | `false`<br> |
