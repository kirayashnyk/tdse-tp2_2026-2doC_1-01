# Actividad 02 - Módulo Sensor (3 Sensor Statechart)

## Registro de Datos en Depuración (Debug)

A continuación se presentan los valores observados en el arreglo de la estructura `task_sensor_dta_list` para las tres instancias de los botones durante la ejecución del programa en reposo:

| Boton | Campo (Atributo) | Valor Medido | Unidad / Descripción |
| :--- | :--- | :--- | :--- |
| D2 | `tick` | 0 | ms (milisegundos) |
| | `state` | ST_BTN_UP (0) | Estado actual de la FSM 0 |
| | `event` | EV_BTN_UP (0) | Último evento procesado |
| D4 | `tick` | 0 | ms (milisegundos) |
| | `state` | ST_BTN_UP (0) | Estado actual de la FSM 1 |
| | `event` | EV_BTN_UP (0) | Último evento procesado |
| D6 | `tick` | 0 | ms (milisegundos) |
| | `state` | ST_BTN_UP (0) | Estado actual de la FSM 2 |
| | `event` | EV_BTN_UP (0) | Último evento procesado |

## Descripción del Funcionamiento

Se escaló la implementación de la máquina de estados finitos para soportar 3 instancias independientes de sensores. Para esto, se definieron arreglos de estructuras de configuración (`cfg`) y de datos (`dta`) dimensionados para 3 elementos. 


La ejecución del statechart de los sensores comienza con la función task_sensor_init(), que inicializa el arreglo de datos (dta) para cada uno de los tres sensores (índices 0, 1 y 2). En este estado inicial, todas las variables arrancan en reposo: tick en 0, state en ST_BTN_UP y event en EV_BTN_UP.

A partir de ahí, en las sucesivas ejecuciones del ciclo principal, la función task_sensor_update() evalúa constantemente el estado de cada pin mediante un bucle for usando la variable index para iterar sobre los 3 botones.

La evolución de cada variable dentro del statechart es la siguiente:

tick (ms): Es el temporizador del filtro antirrebote (debouncing). Su valor evoluciona cada vez que el código detecta un flanco (cambio de estado). Si el botón se presiona (o se suelta), el tick comienza a incrementarse en cada ciclo hasta alcanzar el valor máximo configurado (DEL_BTN_MAX, que suele ser 50 ms). Una vez validado el evento, el temporizador se reinicia a 0.

state: Representa el estado actual dentro de la máquina de estados finitos (FSM) de ese sensor en particular. Evoluciona pasando desde el estado de reposo (ST_BTN_UP) hacia un estado transitorio (ST_BTN_FALLING) cuando se detecta que el botón fue presionado. Si el temporizador (tick) confirma que no es un ruido, el estado evoluciona a ST_BTN_DOWN. Al soltar el botón, ocurre el proceso inverso pasando por el estado transitorio ST_BTN_RISING.

event: Almacena el último evento validado por el statechart. Esta variable solo evoluciona cuando el temporizador de debouncing finaliza exitosamente. Si la máquina de estados confirma la pulsación, cambia a EV_BTN_DOWN. Cuando el botón se suelta y se valida el rebote de subida, vuelve a EV_BTN_UP. Este es el evento que, tras ser validado, se encola y se envía al siguiente módulo (System) mediante la función put_event_task_system().


**Nota de Implementación:** Cabe recalcar que en el código fuente se mantuvo habilitado el botón integrado de la placa (`BTN_A` / B1 Blue Push Button). Este pulsador se dejó configurado ejecutándose en paralelo a los 3 sensores externos, utilizándose únicamente como herramienta de diagnóstico para confirmar el correcto funcionamiento general del programa y de la cola de eventos.
