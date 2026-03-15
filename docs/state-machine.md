# State Machine — PROFIsafe Connection

## States

| State | Enum | Description |
|-------|------|-------------|
| `POWER_ON` | 0 | Initial state after `profisafe_init()`.  No communication. |
| `F_PAR_REQ` | 1 | F-Host has sent F-Parameter request.  Waiting for F-Device to validate. |
| `F_PAR_ACK` | 2 | F-Device has validated F_Par_CRC.  Waiting for F-Host to confirm. |
| `TOGGLE_INIT` | 3 | First toggle exchange in progress.  Verifying round-trip. |
| `DATA_EXCHANGE` | 4 | Normal operation.  CRC-protected safety telegrams flowing. |
| `PASSIVATION` | 5 | Safety violation detected.  Fail-safe values active. |

## Transition Table

| From | To | Trigger | Guard | Action |
|------|----|---------|-------|--------|
| `POWER_ON` | `F_PAR_REQ` | `profisafe_init()` returns OK | Config valid, buffers allocated | Arm watchdog, initialise SeqNr to 0, clear diag counters |
| `F_PAR_REQ` | `F_PAR_ACK` | RX telegram with F-Parameters | F_Par_CRC matches computed CRC | Store F-Parameters, seed CRC1/CRC2 with codename |
| `F_PAR_REQ` | `PASSIVATION` | Watchdog expires | — | Set `CE_Timeout`, activate fail-safe values |
| `F_PAR_ACK` | `TOGGLE_INIT` | F-Host receives `F_Par_CRC_ok=1` | Status byte bit 1 is set | Set initial toggle value, send first data telegram |
| `F_PAR_ACK` | `PASSIVATION` | F_Par_CRC mismatch on retry | Retry count > 3 | Set `Device_Fault`, log diag event |
| `F_PAR_ACK` | `PASSIVATION` | Watchdog expires | — | Set `CE_Timeout` |
| `TOGGLE_INIT` | `DATA_EXCHANGE` | Toggle bit mirrored correctly | `Toggle_h == Toggle_d` (F-Device view) | Reset watchdog, begin SeqNr monitoring |
| `TOGGLE_INIT` | `PASSIVATION` | Toggle mismatch | `Toggle_h != expected` after 2 attempts | Set toggle error flag |
| `TOGGLE_INIT` | `PASSIVATION` | Watchdog expires | — | Set `CE_Timeout` |
| `DATA_EXCHANGE` | `DATA_EXCHANGE` | Valid telegram received | CRC1 OK, CRC2 OK, SeqNr in window, toggle OK | Re-arm watchdog, update safety data, increment SeqNr |
| `DATA_EXCHANGE` | `PASSIVATION` | CRC1 mismatch | — | Set `CE_CRC`, increment `diag.crc1_errors` |
| `DATA_EXCHANGE` | `PASSIVATION` | CRC2 mismatch | — | Set `CE_CRC`, increment `diag.crc2_errors` |
| `DATA_EXCHANGE` | `PASSIVATION` | SeqNr outside tolerance window | `|rx_seqnr - expected| > tolerance` | Set `CE_SeqNr`, increment `diag.seqnr_errors` |
| `DATA_EXCHANGE` | `PASSIVATION` | Watchdog timeout | No valid telegram for `watchdog_time_ms` | Set `CE_Timeout`, increment `diag.timeouts` |
| `DATA_EXCHANGE` | `PASSIVATION` | Toggle bit mismatch | `Toggle_h != Toggle_d` | Increment `diag.toggle_errors` |
| `DATA_EXCHANGE` | `PASSIVATION` | `profisafe_passivate()` called | Application requests passivation | Immediate fail-safe, log reason |
| `PASSIVATION` | `F_PAR_REQ` | `profisafe_reset()` called | Operator / host command received | Clear all errors, re-init SeqNr, re-arm watchdog |

## Passivation Behaviour

When the state machine enters `PASSIVATION`:

1. **Safety output data** is replaced with **fail-safe values** (all zeros by default, configurable via F-Parameters).
2. The `FV_activated` bit is set in the status byte of all subsequent TX telegrams.
3. CRC1 and CRC2 continue to be computed over the fail-safe data — the partner sees valid telegrams with fail-safe content.
4. The state machine will **not** leave `PASSIVATION` without an explicit `profisafe_reset()` call.
5. `profisafe_reset()` is typically gated by operator acknowledgement (key switch, HMI button, or host controller command).

## Timing Constraints

| Constraint | Value | Rationale |
|-----------|-------|-----------|
| Minimum watchdog time | 16 ms (1 cycle) | Cannot be less than the communication cycle |
| Maximum watchdog time | 65535 ms | 16-bit millisecond counter |
| Toggle mirror deadline | Within 1 cycle | F-Host must mirror `Toggle_d` before the next F-Device TX |
| Passivation latency | < 1 cycle | Detection and fail-safe output happen in the same `process_rx` / `build_tx` call pair |
| F-Parameter exchange timeout | 3 × watchdog time | Allows 3 retransmission attempts before giving up |

## Sequence Diagram — Normal Connection Setup

```
  F-Host                           F-Device
    │                                 │
    │  ──── F_Par_Req=1 ──────────►  │   [F_PAR_REQ]
    │        (F-Parameters in data)   │
    │                                 │
    │                           validate F_Par_CRC
    │                                 │
    │  ◄─── F_Par_CRC_ok=1 ────────  │   [F_PAR_ACK]
    │                                 │
    │  ──── Toggle_h=0 ───────────►  │   [TOGGLE_INIT]
    │  ◄─── Toggle_d=0 ────────────  │
    │                                 │
    │  ──── Toggle_h=1 ───────────►  │   [DATA_EXCHANGE]
    │  ◄─── Toggle_d=1 ────────────  │
    │       (safety data + CRC)       │
    │                                 │
    │  ──── Toggle_h=0 ───────────►  │   continuing ...
    │  ◄─── Toggle_d=0 ────────────  │
    │       ...                       │
```

## Sequence Diagram — Passivation on CRC Error

```
  F-Host                           F-Device
    │                                 │
    │  ──── valid telegram ────────►  │   [DATA_EXCHANGE]
    │  ◄─── valid telegram ─────────  │
    │                                 │
    │  ──── CORRUPTED telegram ───►  │   ← bit flip in transit
    │                                 │
    │                           CRC1 check FAILS
    │                           → enter PASSIVATION
    │                                 │
    │  ◄─── FV_activated=1 ─────────  │   [PASSIVATION]
    │       (fail-safe values)        │
    │                                 │
    │  (F-Host detects partner        │
    │   passivation, also             │
    │   enters PASSIVATION)           │
```
