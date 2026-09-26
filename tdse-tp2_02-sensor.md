# Actividad 02 - Módulo Sensor (3 Sensor Statechart)

## Registro de Datos en Depuración (Debug)

A continuación se presentan los valores observados en el arreglo de la estructura `task_sensor_dta_list` para las tres instancias de los botones durante la ejecución del programa en reposo:

| Sensor (Índice) | Campo (Atributo) | Valor Medido | Unidad / Descripción |
| :--- | :--- | :--- | :--- |
| D2 | `tick` | 0 | ms (milisegundos) |
| | `state` | ST_BTN_UP (0) | Estado actual de la FSM 0 |
| | `event` | EV_BTN_UP (0) | Último evento procesado |
| D4 | `tick` | 0 | ms (milisegundos) |
| | `state` | ST_BTN_UP (0) | Estado actual de la FSM 1 |
| | `event` | EV_BTN_UP (0) | Último evento procesado |
| D7 | `tick` | 0 | ms (milisegundos) |
| | `state` | ST_BTN_UP (0) | Estado actual de la FSM 2 |
| | `event` | EV_BTN_UP (0) | Último evento procesado |

## Descripción del Funcionamiento

Se escaló la implementación de la máquina de estados finitos para soportar 3 instancias independientes de sensores. Para esto, se definieron arreglos de estructuras de configuración (`cfg`) y de datos (`dta`) dimensionados para 3 elementos. 

La función `task_sensor_update()` ahora evalúa y actualiza el statechart de cada botó



Pines D2, D4 y D7
