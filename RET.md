# RET — Timing Evidence Report

**Team:** Maria de los Ángeles Prieto Ortega · **Boards:** ESP32-C6 SuperMini (`SuperMini-C6-Maria`)

**Living document** — updated every week; handed in at the workshop (week 8) and
at the close (week 16).

**House rule:** *"show me the trace"* — every timing claim cites a measurement.

## 1. The system and its task set

Requirements first — one sentence each, EARS style (*when/while `<condition>`, the
system shall `<response>` within `<deadline>`*) — then the task that implements them:

| ID | Requirement |
| --- | --- |
| REQ-CTRL-01 | While the system is irrigating, the control loop shall regulate flow and pressure every 10 ms, with its deadline equal to its period. |
| REQ-SAFE-01 | When an overpressure condition is detected, the system shall stop the valve or pump within 5 ms. |
| REQ-SENS-01 | While the controller is active, the system shall sample its sensors at 1 kHz with bounded jitter. |
| REQ-TEL-01 | While the controller is operating, the system shall transmit telemetry to the Hub; delayed data may degrade service but does not cause system failure. |
| REQ-CONS-01 | When a console command is received, the system shall respond before the command becomes obsolete; a late response is worthless but harmless. |

| Task | Req. | Type (H/F/S) | Period | Deadline | Measured C_i | How it was measured |
| --- | --- | --- | --- | --- | --- | --- |
| Flow/pressure control loop | REQ-CTRL-01 | Hard | 10 ms | 10 ms (= T) | ____ | Pending measurement |
| Overpressure emergency stop | REQ-SAFE-01 | Hard | Event-driven | < 5 ms | ____ | Pending measurement |
| Sensor sampling | REQ-SENS-01 | Hard | 1 ms (1 kHz) | ≤ 1 ms | ____ | Pending measurement |
| Telemetry to the Hub | REQ-TEL-01 | Soft | TBD | TBD | ____ | Pending measurement |
| Command console | REQ-CONS-01 | Firm | Event-driven | TBD | ____ | Pending measurement |

## 2. ADRs

### ADR-001 — `<title>`

**Context:** … · **Decision:** … · **Justification (with numbers):** … · **Status:** …

## 3. Evidence by week

Each entry cites the `REQ`(s) it verifies.

### Week 2 — superloop baseline (state the board)

<jitter/latency table + a one-sentence reading>

### Week 3 — S3 baseline and silicon comparison

…

## 4. Schedulability analysis

U = ΣC_i/T_i with **measured** C_i; test used (RM / hyperbolic / EDF); RTA as a
script with its output; blocking B_i if there are mutexes. (Formulas: READINGS.md,
"The math that does get used".)

## 5. Functional safety (final project)

Declared safe state (failure ⇒ valve closed), watchdog, and the evidence of the
fail-safe firing.
