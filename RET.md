# RET — Timing Evidence Report

**Team:** Maria de los Ángeles Prieto Ortega (trabajo individual)  
**Boards:** ESP32-C6 SuperMini (`SuperMini-C6-Maria`)  
**Zephyr target:** `esp32c6_devkitc/esp32c6/hpcore`  
**Simulation target:** `native_sim/native/64`

**Living document** — updated every week; handed in at the workshop (week 8) and
at the close (week 16).

**House rule:** *"show me the trace"* — every timing claim cites a measurement.

> **Platform adaptation:** The laboratory reference platform is the
> STM32C0116-DK and the project scenario proposes an ESP32-S3. For this
> implementation, the available ESP32-C6 SuperMini was used. A devicetree
> overlay defines the conventional onboard LED on GPIO15 and routes the Zephyr
> console through the ESP32-C6 USB Serial/JTAG controller.

## 1. The system and its task set

Requirements are written first using the EARS style (*when/while a condition
holds, the system shall provide a response within a deadline*), followed by the
tasks that implement them.

| ID | Requirement |
| --- | --- |
| REQ-CTRL-01 | While the system is irrigating, the system shall execute the flow and pressure control loop every 10 ms, with its deadline equal to its period. |
| REQ-SAFE-01 | When an overpressure condition is detected, the system shall stop the valve or pump within 5 ms. |
| REQ-SENS-01 | While the controller is active, the system shall sample its sensors at 1 kHz with bounded jitter. |
| REQ-TEL-01 | While the controller is operating, the system shall transmit telemetry to the Hub within a configured interval; delayed telemetry may degrade service but does not cause system failure. |
| REQ-CONS-01 | When a console command is received, the system shall respond before the configured response deadline; a response received after that deadline is worthless but harmless. |

| Task | Req. | Type (H/F/S) | Period | Deadline | Measured C_i | How it was measured |
| --- | --- | --- | --- | --- | --- | --- |
| Flow/pressure control loop (PWM to valve/pump) | REQ-CTRL-01 | H (Hard) | 10 ms | 10 ms (= T) | ____ | GPIO + logic analyzer / trace (pending) |
| Overpressure emergency stop | REQ-SAFE-01 | H (Hard) | Event-driven | < 5 ms | ____ | GPIO + logic analyzer / trace (pending) |
| Sensor sampling | REQ-SENS-01 | H (Hard) | 1 ms (1 kHz) | TBD; jitter must be bounded | ____ | GPIO + logic analyzer / trace (pending) |
| Telemetry to the Hub | REQ-TEL-01 | S (Soft) | TBD | TBD | ____ | Trace and timestamp analysis (pending) |
| Command console | REQ-CONS-01 | F (Firm) | Event-driven | TBD | ____ | Serial trace and timestamps (pending) |

> `C_i` is intentionally left as `____` until execution time is measured in a
> later laboratory. No WCET value is inferred from build or flash time.

## 2. ADRs

### ADR-001 — Use the ESP32-C6 SuperMini as the laboratory controller

**Context:** The laboratory specifies an STM32C0116-DK, while the available
hardware is an ESP32-C6 SuperMini. The project requires continued use of Zephyr
and later timing measurements.

**Decision:** Use the ESP32-C6 SuperMini with the Zephyr board target
`esp32c6_devkitc/esp32c6/hpcore`. Add a devicetree overlay for the GPIO15 LED and
the USB Serial/JTAG console.

**Justification (with numbers):** Blinky was compiled and flashed successfully;
the modified serial application operated at 115200 baud, 8 data bits, no parity
and 1 stop bit. The measured incremental build-to-flash cycle was 2.618 s, below
the stakeholder objective of 60 s.

**Status:** Accepted for the course laboratories.

## 3. Evidence by week

Each entry cites the requirement or bring-up capability it verifies. The
screenshots and video should be stored in `evidence/week01/`.

### Week 1 — board and toolchain bring-up

| Evidence | Result | Artifact |
| --- | --- | --- |
| E01 | Ubuntu/UTM detected the Espressif USB Serial/JTAG device and exposed `/dev/ttyACM0`. | `evidence/week01/E01_usb_detection.png` |
| E02 | Zephyr successfully built Blinky for `esp32c6_devkitc/esp32c6/hpcore`. | `evidence/week01/E02_blinky_build.png` |
| E03 | The Blinky image was written and its hash verified by `esptool`. | `evidence/week01/E03_blinky_flash.png` |
| E04 | The conventional onboard LED connected to GPIO15 alternated between on and off. | `evidence/week01/E04_blinky_gpio15.mp4` |
| E05 | The incremental build-to-flash cycle completed in 2.618 s, satisfying the objective of less than 60 s. | `evidence/week01/E05_iteration_time.png` |
| E06 | The USB serial console operated at 115200 8N1 and displayed the modified message `Lab 1 - Maria - ESP32-C6 SuperMini OK`. | `evidence/week01/E06_modified_serial_message.png` |
| E07 | The modified application compiled and executed using `native_sim/native/64` on Ubuntu AArch64. | `evidence/week01/E07_native_sim.png` |
| E08 | This RET was committed and pushed to the team repository. | Git commit and repository history |

**Week 1 reading:** The complete build→flash cycle took 2.618 s, which is
57.382 s below the 60 s stakeholder limit. Both the physical ESP32-C6 SuperMini
and the 64-bit native simulator executed the application successfully.

### Week 2 — superloop baseline (state the board)

<jitter/latency table + a one-sentence reading>

### Week 3 — S3 baseline and silicon comparison

…

## 4. Schedulability analysis

U = ΣC_i/T_i with **measured** C_i; test used (RM / hyperbolic / EDF); RTA as a
script with its output; blocking B_i if there are mutexes. (Formulas:
`READINGS.md`, "The math that does get used".)

## 5. Functional safety (final project)

Declared safe state (failure ⇒ valve closed), watchdog, and the evidence of the
fail-safe firing.
