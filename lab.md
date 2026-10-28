# RET — Informe de Evidencia de Tiempos

**Grupo:** `María Prieto Ortega` `Rafael Torres Choperena` `Mariana Zuluaga Yepes`  ·  **Placa:** `Nucleo-L476RG` - `ESP32-C6 Super Mini`

## 1. El sistema y su conjunto de tareas 

| ID | Requisito |
|---|---|
| REQ-SAMP-01 | Mientras el sistema está activo, deberá ejecutar el muestreo de presión y e-stop cada 1 ms, dentro de un plazo de 1 ms. |
| REQ-CTRL-01 | Mientras el sistema está irrigando, deberá ejecutar el control cada 10 ms, dentro de un plazo de 10 ms. |
| REQ-CONS-01 | Cuando el operador ingrese un comando válido, el sistema deberá procesarlo y mostrar la respuesta correspondiente en la consola. |
| REQ-TEL-01 | Mientras el sistema está activo, deberá publicar telemetría cada 1 s. |
| REQ-HMI-01 | Mientras la HMI está habilitada, deberá actualizar la pantalla cada 500 ms. |
| REQ-FLOW-01 | Cuando se acumulen 100 pulsos de flujo, deberá procesar el lote en la siguiente iteración disponible del superloop. |
| REQ-SAFE-01 | Si se presenta una sobrepresión o se activa la parada de emergencia, el sistema deberá cerrar la válvula para garantizar la seguridad. |

### Tareas y tiempos de ejecución en Nucleo-L476RG (Lab 2)

| Tarea | Req. | Tipo | Período / activación | Fecha límite | `C_i` observado | Método |
|---|---|---|---|---|---:|---|
| Muestreo | REQ-SAMP-01 | Duro | 1 ms | 1 ms | 5 µs | Ancho de pulso D3, Logic 2 |
| Control | REQ-CTRL-01 | Duro | 10 ms | 10 ms | 2.5 µs | Ancho de pulso D4, Logic 2 |
| Consola (`help`) | REQ-CONS-01 | Firme | Por evento | No definida | 1.313 ms | Ancho de pulso D5, Logic 2 |
| Telemetría | REQ-TEL-01 | Suave | 1 s | 1 s | 6.1555 ms | Ancho de pulso D6, Logic 2 |
| Lote de flujo | REQ-FLOW-01 | Firme | Cada 100 pulsos | No definida | 70.5 µs | Ancho de pulso D7, Logic 2 |
| Pantalla OLED | REQ-HMI-01 | Suave | 500 ms | 500 ms | 25.06 ms | Ancho de pulso D8, Logic 2 |

### Tareas y tiempos de ejecución en ESP32-C6 (Lab 3)

| Tarea | Requisito | Tipo | Período | Deadline | `C_i` máximo medido |
|---|---|---|---:|---:|---:|
| Muestreo | REQ-SAMP-01 | Duro | 1 ms | 1 ms | 12,937 µs |
| Control PI | REQ-CTRL-01 | Duro | 10 ms | 10 ms | 34,500 µs |
| Consola (`calib`) | REQ-CONS-01 | Firme | Esporádica | No definido | 408,623 ms |
| Telemetría USB | REQ-TELE-01 | Suave | 1 s | Flexible | 50,442 ms |
| Lote de flujo | REQ-FLOW-01 | Suave | Cada 100 pulsos | No definido | No medido |
| Pantalla OLED | REQ-HMI-01 | Suave | 500 ms | 500 ms | No medida en C6 |

## Tareas Laboratorios

### Diagrama del superloop — Lab 2, tarea A

```mermaid
flowchart TD
    T["Temporizador cada 1 ms: ISR"] --> P["Incrementa ticks_pending y actualiza backlog_peak"]
    F["Flanco de flujo D9 / PC7: ISR"] --> Q["Incrementa flow_pulses"]

    P --> L["Superloop: while (1)"]

    L --> C["Lee consola D5 / PB4"]
    C --> H["Actualiza pantalla OLED D8 / PA9"]
    H --> E["Envía telemetría D6 / PB10"]
    E --> D{"¿Hay tick pendiente?"}

    D -- Sí --> S["Muestrea presión y e-stop D3 / PB3"]
    S --> K{"¿Van 10 muestras?"}
    K -- Sí --> R["Control D4 / PB5 y válvula PA5"]
    K -- No --> B["Revisa y procesa lote de flujo D7 / PA8"]
    R --> B
    D -- No --> B

    Q --> B
    B --> L
```

## Lectura del flujo

- El temporizador genera una interrupción cada 1 ms. La ISR no ejecuta el muestreo: incrementa `ticks_pending` y actualiza `backlog_peak` si se acumulan ticks.

- El superloop recorre siempre el mismo orden: consola, pantalla OLED y telemetría. Después revisa si hay un tick de muestreo pendiente.

- Si existe un tick pendiente, ejecuta el muestreo de presión y e-stop en D3. Cada diez muestras ejecuta además el control en D4 y actualiza la válvula en PA5.

- La entrada de flujo llega por D9/PC7. Su ISR incrementa `flow_pulses`; el superloop revisa ese contador en la tarea de lote de flujo, instrumentada en D7.

- La tarea de flujo se revisa en cada vuelta del ciclo. Cuando no hay tick pendiente, el superloop pasa directamente desde la telemetría hacia la revisión de flujo.

- Como todas las tareas comparten un solo `while (1)`, una tarea lenta bloquea temporalmente a las demás. La actualización de OLED y el comando `calib` pueden retrasar el muestreo y acumular `ticks_pending`.

## 2. ADRs

### ADR-001 — Arquitectura del nodo con kernel multihilo

**Contexto:** En la línea base del superloop de la ESP32-C6, el comando
`calib` bloqueó muestreo y control durante aproximadamente 408 ms. El jitter
máximo de muestreo alcanzó 408,297 ms, el período máximo de control alcanzó
416,317 ms y se acumularon 409 ticks.

**Decisión:** Migrar todas las tareas del superloop a entidades del kernel de
Zephyr. Muestreo se ejecuta en `sampling_thread`; control en
`control_thread`; flujo se procesa en la cola de trabajo privada `flow_wq`;
pantalla, telemetría y consola se ejecutan en sus propios hilos. `main`
solamente inicializa el hardware, inicia el temporizador y retorna.

**Justificación:** En la versión con kernel, incluso durante `calib`, el jitter
máximo de muestreo fue 7 µs, el período máximo de control fue 10,005 ms y
`backlog_peak` permaneció en 0. Esto elimina el bloqueo de aproximadamente
408 ms que afectaba las tareas críticas en el superloop.

**Costo:** La transición desde la liberación de control hasta su inicio tuvo un
máximo de 15 µs. Como control se libera 100 veces por segundo, el costo
estimado es `100 × 15 µs = 1.500 µs/s = 1,5 ms/s`.

**Estado:** Aceptada e implementada.

## 3. Evidencia por semana

### Semana 2 — Línea base del superloop  
**Requisitos verificados:** REQ-SAMP-01, REQ-CTRL-01, REQ-FLOW-01.

**Placa:** Nucleo-L476RG  
**Instrumento:** Logic 2  
**Señales:** D3 muestreo, D4 control, D5 consola, D6 telemetría,
D7 lote de flujo, D8 OLED y D9 entrada de flujo.

| **Medición** | **Valor obtenido** | **Referencia** |
|---|---:|---|
| Período real de muestreo, promedio | **999 µs** | 1000 µs |
| Jitter máximo de muestreo durante ≥ 30 s | **22 870 µs** | Línea base con OLED y caché |
| Latencia ISR → servicio del superloop, pulso de flujo | **12.5 µs** | D9 → D7 |
| Jitter máximo de muestreo con la caché flash desactivada | **24 580 µs** | Aumentó ≈ 1710 µs frente a la línea base |
| Jitter de muestreo con el comando bloqueante activo | **431 332 µs** | Comando `calib` |
| `backlog_peak`, inactivo → durante el comando bloqueante | **30 → 432 ticks** | Contador interno del firmware |

> **Criterio usado para jitter máximo tardío:**
>
> `jitter = período máximo observado − 1000 µs`
>
> Este valor indica cuánto se retrasó la ejecución de muestreo frente a su
> período nominal de 1 ms.

La frecuencia media se mantuvo cercana a 1 kHz, pero la actualización de la OLED produjo huecos de hasta 25.58 ms en D3. El comando `calib` bloqueó el único superloop durante 408.882 ms, acumuló 432 ticks y retrasó tanto el muestreo como el control.

### Análisis de causa raíz del comando `calib`

En `firmware/superloop/src/main.c:409`, la función `cmd_calib()` ejecuta
`k_busy_wait(400)` dentro de un ciclo de 1000 rondas. Esto bloquea el
superloop aproximadamente 400 ms. El comando se invoca desde
`console_handle()` en la línea 434 y la consola se ejecuta dentro del único
`while (1)` en la línea 503. Mientras la calibración no retorna, el programa
no atiende `ticks_pending`; por ello se observó un período máximo de D3 de
432.332 ms y `backlog_peak=432`.

### Semana 3 — Comparación de la línea base del C6 y el silicio  
**Requisitos verificados:** REQ-SAMP-01 y REQ-CTRL-01.

Aunque la guía está diseñada para el ESP32-S3, en este caso se utilizó un
ESP32-C6 Super Mini. Sin embargo, el objetivo de la práctica se mantiene:
realizar el port mediante devicetree y trasladar el muestreo a un hilo de
ejecución.

| Medición | Nucleo Lab 2 | C6 superloop | C6 + `sampling thread` |
|---|---:|---:|---:|
| Jitter máximo normal | 22,870 ms | 50,446 ms | 2 µs |
| Período máximo normal | 23,870 ms | 51,446 ms | 1,001 ms |
| Jitter con `calib` | 431,332 ms | 408,297 ms | 2 µs |
| Período máximo con `calib` | 432,332 ms | 409,297 ms | 1,001 ms |
| `backlog_peak` con `calib` | 432 ticks | 409 ticks | 0 ticks |
| `lat_peak_us` | — | — | 9 µs |
| Período máximo de control con `calib` | — | 416,317 ms | 420,004 ms |
| Duración de `calib` | 408,882 ms | 407,482 ms | 408,516 ms |

#### Evidencia del port

El port se realizó mediante
`firmware/superloop/boards/esp32c6_devkitc_hpcore.overlay`. El archivo define
los GPIO de la ESP32-C6, la entrada de flujo, la válvula, las señales de
instrumentación y la consola USB Serial/JTAG.

La aplicación se compiló, se grabó en la placa y respondió al comando `status`.

El primer commit de la adaptación a ESP32-C6 fue `8e72f6f`:

.../esp32c6_devkitc_hpcore.overlay | 12 +++++
firmware/superloop/src/main.c       | 37 ++++++++++++++--
2 files changed, 46 insertions(+), 3 deletions(-)

#### Comparación de silicios

Aunque la ESP32-C6 funciona a una frecuencia mayor que la Nucleo-L476RG, su
superloop tuvo mayor jitter máximo normal: 50,446 ms frente a 22,870 ms. La frecuencia de CPU no determina por sí sola el peor caso: la consola USB,
`printk`, la ejecución desde flash con caché y las tareas bloqueantes influyen
en el tiempo máximo observado.

La comparación es indicativa: la Nucleo se midió con analizador lógico y la C6
con instrumentación interna mediante `k_cycle_get_32()`.

#### Sampling frente a control durante `calib`

Durante `calib`, el hilo de muestreo mantuvo jitter máximo de 2 µs, período
máximo de 1,001 ms y `backlog_peak=0`; por ello cumple REQ-SAMP-01.

Control permaneció en el superloop y alcanzó un período máximo de 420,004 ms,
mayor que su deadline de 10 ms. Por tanto, REQ-CTRL-01 no se cumple durante
`calib`.

#### Estado compartido

| Variable | Escritores | Lectores |
|---|---|---|
| `pressure_mv` | Hilo de muestreo | Control, telemetría, pantalla y `status` |
| `estop` | Hilo de muestreo y comando `clear` | Control, telemetría, pantalla y `status` |
| `setpoint_mv` | Joystick y comando `set` | Control, telemetría, pantalla y `status` |

El programa funciona en esta prueba porque la C6 tiene un solo núcleo principal
y usa enteros alineados, pero en C todavía pueden existir carreras de datos.

### Semana 4 — Migración completa al kernel y comparación A/B  
**Requisitos verificados:** REQ-SAMP-01 y REQ-CTRL-01.

En esta semana se completó la migración del superloop a Zephyr. Las tareas se
organizaron por prioridad: `sampling_thread` (prioridad 2), `control_thread`
(3), `flow_wq` (5), `display_thread` (7), `telemetry_thread` (8) y
`console_thread` (9).

La ISR del temporizador publica las liberaciones de muestreo en `tick_q`.
Cada décima muestra entrega el semáforo `control_sem` para liberar control. La
ISR de flujo programa `task_flow_batch()` en una cola de trabajo privada. Cada
hilo bloquea mientras no tiene trabajo, por lo que no consume CPU esperando.

| Medición (ESP32-C6) | C6 superloop, Semana 3 | Predicción, kernel | Kernel, Semana 4 |
|---|---:|---:|---:|
| Jitter máximo de muestreo | 50,446 ms | ≤ 10 µs | 7 µs |
| Jitter máximo de muestreo con `calib` | 408,297 ms | ≤ 10 µs | 7 µs |
| Período máximo de control con `calib` | 416,317 ms | ≈ 10 ms | 10,005 ms |
| Latencia ISR → hilo (`lat_peak_us`) | — | ≤ 20 µs | 12 µs |
| Brecha interna de liberación a inicio de control | — | ≤ 20 µs | 15 µs |
| Uso máximo de pila de `control_tid` | — | < 1024 bytes | 236 bytes / 1024 bytes asignados |

La brecha de 15 µs se obtuvo mediante instrumentación interna entre la entrega
de `control_sem` y el inicio de `control_thread`. Es una aproximación al costo
de cambio de contexto; no equivale a una medición externa visible en GPIO.

Las mediciones se realizaron en la misma ESP32-C6 y bajo las mismas
condiciones: ejecución normal por al menos 30 segundos y ejecución del comando
`calib`. Debido a que no se contó con analizador lógico para la C6, los valores
se obtuvieron mediante instrumentación interna con `k_cycle_get_32()`.

Durante la condición normal, el kernel mantuvo `backlog_peak=0`,
`samp_jitter_peak_us=7` y `ctrl_period_peak_us=10005`. Durante `calib`, los
mismos máximos se conservaron: `backlog_peak=0`,
`samp_jitter_peak_us=7` y `ctrl_period_peak_us=10005`.

La migración elimina el bloqueo de aproximadamente 408 ms que producía el
superloop. Con el kernel, `calib` se ejecuta en el hilo de consola de menor
prioridad, mientras muestreo y control continúan ejecutándose cerca de sus
períodos nominales.

La evidencia muestra que el bloqueo dejó de causar incumplimientos de cientos
de milisegundos. Sin embargo, el período máximo de control fue 10,005 ms, es
decir, 5 µs superior al deadline estricto de 10 ms. Por tanto, la medición
demuestra una mejora sustancial, pero no constituye por sí sola una prueba
formal de cumplimiento estricto de REQ-CTRL-01.

#### Costo del cambio de contexto

Cada 10 muestras, `sampling_thread` libera `control_thread` mediante
`control_sem` y luego vuelve a bloquearse. La brecha máxima medida entre la
liberación y el inicio de control fue 15 µs.

Control se libera una vez cada 10 ms, es decir, 100 veces por segundo:

`100 liberaciones/s × 15 µs = 1.500 µs/s = 1,5 ms/s`.

Por tanto, el costo máximo estimado de estas transiciones es 1,5 ms por segundo
de ejecución, equivalente a aproximadamente 0,15 % de un segundo.

#### Marcas de uso máximo de pila

El comando `threads`, que ejecuta `thread_analyzer_print(0)`, produjo los
siguientes valores bajo ejecución con telemetría, consola, pantalla y `calib`.

| Hilo o cola de trabajo | Pila usada | Pila asignada | Uso |
|---|---:|---:|---:|
| `sampling_tid` | 252 bytes | 1536 bytes | 16 % |
| `control_tid` | 236 bytes | 1024 bytes | 23 % |
| `flow_wq` | 268 bytes | 1024 bytes | 26 % |
| `display_tid` | 220 bytes | 2048 bytes | 10 % |
| `telemetry_tid` | 660 bytes | 2048 bytes | 32 % |
| `console_tid` | 804 bytes | 2048 bytes | 39 % |

El hilo más cercano a su límite fue `console_tid`, con 804 de 2048 bytes
utilizados (39 %). Ninguna pila mostró riesgo de desbordamiento en esta prueba.

## 4. Análisis de planificabilidad

U = ΣC_i/T_i with **measured** C_i; test used (RM / hyperbolic / EDF); RTA as a
script with its output; blocking B_i if there are mutexes. (Formulas: READINGS.md,
"The math that does get used".)

## 5. Seguridad funcional (final project)

Declared safe state (failure ⇒ valve closed), watchdog, and the evidence of the
fail-safe firing.



