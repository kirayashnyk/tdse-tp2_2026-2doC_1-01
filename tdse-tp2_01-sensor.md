# Actividad 01 - Modulo Sensor (Statechart Debouncing)

## Registro de Datos en Depuración (Debug)

A continuación se presentan los valores observados en la estructura `task_sensor_dta_list[0]` durante la ejecución del programa en reposo:

| Campo (Atributo) | Valor Medido | Unidad / Descripción |
| :--- | :--- | :--- |
| `tick` | 0 | ms (milisegundos) |
| `state` | ST_BTN_UP (0) | Estado actual de la FSM |
| `event` | EV_BTN_UP (0) | Último evento procesado |

## Descripción del Funcionamiento
Se implementó la máquina de estados finitos con filtrado antirrebote (*debouncing*) de 50 ms. Al presionar o soltar el botón, el sistema transita correctamente por los estados intermedios `ST_BTN_FALLING` y `ST_BTN_RISING` antes de confirmar el evento y enviarlo al módulo `System`.
