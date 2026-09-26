# Actividad 05 - Módulo Actuator (Escalamiento a Múltiples LEDs)

## Descripción del Funcionamiento

El objetivo de esta etapa es escalar el modelo del actuador para soportar múltiples instancias independientes, aplicando el mismo concepto utilizado para los sensores en la Actividad 02. El sistema evoluciona de controlar un único LED a manejar un arreglo de actuadores, permitiendo, por ejemplo, simular un semáforo de acceso con un indicador de "barrera abriendo/abierta" (LED Verde) y otro de "barrera cerrando/cerrada" (LED Rojo).

Para lograr esta concurrencia, el código define arreglos para la configuración (`task_actuator_cfg_list`) y para los datos dinámicos (`task_actuator_dta_list`). La función `task_actuator_update()` utiliza un bucle `for` para evaluar e iterar sobre el statechart de cada LED por separado en cada ciclo del programa.

### Integración y Comportamiento Esperado (Semáforo de Barrera)

El módulo System ahora envía los eventos a la cola del actuador acompañados de un identificador único (ID) para rutear el comando al LED correcto (ej. `ID_LED_BARRIER_OPEN` o `ID_LED_BARRIER_CLOSE`). El comportamiento visual del hardware, gestionado por la máquina de estados, sigue esta secuencia:

1.  **Reposo (Vehículo en espera):** El LED de barrera cerrada (Rojo) se encuentra en estado `ST_LED_ON` (encendido fijo), indicando que no se puede avanzar. El LED de barrera abierta (Verde) permanece en `ST_LED_OFF`.
2.  **Transición de Apertura:** Al presionar el botón de ingreso, el módulo System despacha un evento `EV_LED_OFF` dirigido al LED Rojo (apagándolo) y un evento `EV_LED_BLINK` al LED Verde. El LED Verde parpadea mientras transcurre el tiempo mecánico de subida de la barrera.
3.  **Paso Habilitado:** Una vez finalizado el tiempo de apertura, el sistema envía un evento `EV_LED_ON` al LED Verde, el cual pasa a estar encendido de forma continua, habilitando el paso del auto.
4.  **Transición de Cierre:** Cuando el vehículo libera el sensor de presencia (bobina), el sistema ordena apagar el LED Verde (`EV_LED_OFF`) y hace parpadear el LED Rojo (`EV_LED_BLINK`) como advertencia mientras la barrera desciende. Al completarse el tiempo, el LED Rojo vuelve a quedar encendido fijo, reiniciando el ciclo.
