# Architecture

## Layer Model

The PROFIsafe telegram stack is structured as a strict layered architecture.  Each layer communicates only with its immediate neighbours through well-defined function-call interfaces.  No layer reaches past its neighbour — this is enforced by the module dependency rules in the internal SIL 3 coding standard (rule 8: modular architecture with controlled coupling).

```
 ┌──────────────────────────────────────────────────────────────┐
 │                     SAFETY APPLICATION                       │
 │                                                              │
 │  Responsible for:                                            │
 │  • Mapping physical signals to safety process data           │
 │  • Implementing STO, SS1, SLS, SSM safety functions          │
 │  • Evaluating group status and activation/blocking           │
 │                                                              │
 │  Interface:                                                  │
 │    profisafe_get_safety_data() → reads validated RX payload  │
 │    profisafe_set_safety_data() → writes TX payload           │
 │    profisafe_get_state()       → reads connection state      │
 ├──────────────────────────────────────────────────────────────┤
 │                     PROFISAFE CORE                           │
 │                                                              │
 │  Orchestrates the safety protocol:                           │
 │  • Calls CRC engine, sequence monitor, watchdog, state FSM  │
 │  • Coordinates F-Parameter exchange at connection setup      │
 │  • Decides passivation based on sub-module verdicts          │
 │                                                              │
 │  Interface:                                                  │
 │    profisafe_init()       → allocate + configure channel     │
 │    profisafe_process_rx() → decode + validate received frame │
 │    profisafe_build_tx()   → encode transmit frame            │
 │    profisafe_passivate()  → force immediate passivation      │
 │    profisafe_reset()      → restart connection setup         │
 ├──────────────────────────────────────────────────────────────┤
 │                     SUB-MODULES                              │
 │                                                              │
 │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐ │
 │  │  CRC Engine   │  │  Seq Monitor │  │  Watchdog Timer    │ │
 │  │  crc1_calc()  │  │  seqmon_rx() │  │  wdog_arm()       │ │
 │  │  crc2_calc()  │  │  seqmon_tx() │  │  wdog_check()     │ │
 │  └──────────────┘  └──────────────┘  └────────────────────┘ │
 │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐ │
 │  │  State FSM    │  │  F-Param     │  │  Diagnostics       │ │
 │  │  state_step() │  │  fpar_rx()   │  │  diag_inc()       │ │
 │  │  state_get()  │  │  fpar_build()│  │  diag_get()       │ │
 │  └──────────────┘  └──────────────┘  └────────────────────┘ │
 ├──────────────────────────────────────────────────────────────┤
 │                     TRANSPORT ADAPTER                        │
 │                                                              │
 │  Platform-specific glue code (NOT part of this repo):        │
 │  • PROFINET IO: Extract PROFIsafe container from cyclic      │
 │    PROFINET frame (UDP/IP or RT Class 3 VLAN)                │
 │  • PROFIBUS DP: Extract PROFIsafe container from DP-V0/V1   │
 │    cyclic slot                                               │
 │                                                              │
 │  The stack receives raw bytes via process_rx() and returns   │
 │  raw bytes from build_tx() — it never touches the network.   │
 └──────────────────────────────────────────────────────────────┘
```

## Data Flow — Receive Path

```
  PROFINET / PROFIBUS cyclic frame arrives
      │
      ▼
  Transport adapter extracts PROFIsafe container (N+7 bytes)
      │
      ▼
  profisafe_process_rx(channel, buf, len)
      │
      ├─ 1. Parse Status/Control byte (bit-field decode)
      │
      ├─ 2. Extract CRC1, Consecutive Number, VCN, CRC2
      │
      ├─ 3. Verify CRC2 over (SeqNr + VCN)
      │     └─ FAIL → set CE_CRC, passivate
      │
      ├─ 4. Verify CRC1 over (Status + Safety data)
      │     └─ FAIL → set CE_CRC, passivate
      │
      ├─ 5. Check consecutive number against expected window
      │     └─ FAIL → set CE_SeqNr, passivate
      │
      ├─ 6. Check toggle bit matches expected value
      │     └─ FAIL → passivate
      │
      ├─ 7. Re-arm watchdog timer
      │
      ├─ 8. Copy safety data to validated input buffer
      │
      └─ 9. Return PROFISAFE_OK
```

## Data Flow — Transmit Path

```
  Application calls profisafe_set_safety_data(channel, data, len)
      │
      ▼
  profisafe_build_tx(channel, buf, buf_size)
      │
      ├─ 1. Build Status/Control byte (toggle, FV_activated, faults)
      │
      ├─ 2. Copy safety output data into telegram buffer
      │
      ├─ 3. Increment consecutive number
      │
      ├─ 4. Compute VCN
      │
      ├─ 5. Compute CRC1 over (Status + Safety data), seeded with F_Par_CRC
      │
      ├─ 6. Compute CRC2 over (SeqNr + VCN), seeded with codename
      │
      ├─ 7. Assemble telegram: Status | Data | CRC1 | SeqNr | VCN | CRC2
      │
      └─ 8. Return telegram length
```

## Timing Budget (16 ms cycle)

All sub-module timings measured on XMC4500 @ 120 MHz, flash wait-states = 3, compiled with `-Os`:

| Phase | Worst case | Notes |
|-------|-----------|-------|
| Process RX (decode + all checks) | 20 µs | Includes CRC1 + CRC2 + seqmon |
| Safety function (application) | varies | Not part of this stack |
| Build TX (encode + CRCs) | 15 µs | |
| Watchdog check | 1 µs | DWT cycle counter read + compare |
| State machine step | 2 µs | |
| Diagnostics update | 1 µs | Counter increment only |
| **Total stack overhead** | **~39 µs** | 0.24% of 16 ms budget |

The remaining ~15.96 ms is available to the safety application and other non-safety tasks.

## Memory Map

```
  Flash (code + const):
    profisafe_core.c      2.1 KB
    profisafe_crc.c       1.8 KB   (includes 256-entry CRC1 table)
    profisafe_seqmon.c    0.6 KB
    profisafe_fpar.c      1.2 KB
    profisafe_watchdog.c  0.4 KB
    profisafe_state.c     1.1 KB
    profisafe_diag.c      0.5 KB
    profisafe_types.h     0.0 KB   (header-only, no code generation)
    CRC1 lookup table     0.5 KB   (const, in .rodata)
    CRC2 lookup table     0.2 KB   (const, in .rodata)
    ─────────────────────────────
    Total .text + .rodata  8.4 KB

  RAM (per channel):
    profisafe_channel_t   1.2 KB
      ├─ state (FSM)         4 B
      ├─ config snapshot    48 B
      ├─ rx_safety_data    128 B   (max extended payload)
      ├─ tx_safety_data    128 B
      ├─ f_parameters       64 B
      ├─ seqmon state       24 B
      ├─ watchdog state     16 B
      ├─ diag counters      32 B
      ├─ toggle state        4 B
      └─ padding/alignment  ~780 B → actual 1,228 B

  RAM (global, shared):
    CRC1 table             512 B   (if placed in RAM for speed)
    CRC2 table             256 B
    ─────────────────────────────
    Total shared RAM       768 B   (optional — can stay in flash)
```

## Module Dependency Graph

```
  profisafe_core ──────┬──── profisafe_crc
                       ├──── profisafe_seqmon
                       ├──── profisafe_fpar ──── profisafe_crc
                       ├──── profisafe_watchdog
                       ├──── profisafe_state
                       └──── profisafe_diag

  profisafe_types.h ◄──── (all modules include this)
```

No circular dependencies.  All inter-module calls go downward from `profisafe_core`.  Sub-modules do not call each other except `profisafe_fpar` → `profisafe_crc` (for F_Par_CRC computation).
