# PROFIsafe Telegram Stack

**Safety Communication Layer for Industrial Controllers**

![SIL 3 Compliant](https://img.shields.io/badge/SIL_3-IEC_61508-brightgreen)
![IEC 61784-3](https://img.shields.io/badge/IEC_61784--3--3-PROFIsafe-blue)
![ARM Cortex-M](https://img.shields.io/badge/ARM-Cortex--M4-orange)
![Cycle Time](https://img.shields.io/badge/Cycle-16ms_deterministic-yellow)
![Coverage](https://img.shields.io/badge/Branch_Coverage-98%25-brightgreen)
![Code Size](https://img.shields.io/badge/Code_Size-8.4_KB-lightgrey)

---

## Overview

Full PROFIsafe safety communication stack implementing **IEC 61784-3-3** for F-Device and F-Host communication over PROFINET IO and PROFIBUS DP.

PROFIsafe is the safety extension for PROFINET and PROFIBUS defined by [PROFIBUS & PROFINET International (PI)](https://www.profibus.com/).  It provides fail-safe data exchange between safety controllers (F-Host) and safety field devices (F-Device) by layering safety mechanisms on top of existing "black-channel" transport — meaning the safety integrity is independent of the underlying network.

This stack is designed for **SIL 3 / Category 4** applications per **IEC 61508** and **IEC 62061** and runs bare-metal on ARM Cortex-M4 at deterministic 16 ms cycle times.  It is used in industrial series industrial drive controllers to provide safe speed monitoring, safe torque off (STO), safe stop 1 (SS1), and safe limited speed (SLS) functions.

### What this project does

- Encodes and decodes PROFIsafe telegrams with CRC1/CRC2 integrity protection
- Manages the full connection state machine (power-on → F-Parameter exchange → data exchange → passivation)
- Monitors consecutive numbers, watchdog timeouts, and toggle bits to detect communication faults
- Handles both F-Host (controller side) and F-Device (field device side) roles
- Supports dual-channel redundancy with cross-monitoring for Cat 4 architectures

### What this project does NOT do

- It does **not** implement the PROFINET IO or PROFIBUS DP protocol stacks themselves — those are provided by the platform (Hilscher netX, Anybus, or custom MAC driver)
- It does **not** implement the safety application logic (STO, SS1, SLS) — that sits above this layer

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Safety Application                          │
│         (STO, SS1, SLS, Safe Speed Monitor, Safe Brake)         │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ Group Status  │  │ Activation / │  │  Process Data Mapping │  │
│  │ Evaluation    │  │ Blocking     │  │  (Safety I/O ↔ Bytes) │  │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬───────────┘  │
├─────────┼─────────────────┼──────────────────────┼──────────────┤
│         │       F-Host / F-Device Layer          │              │
│         │                                        │              │
│  ┌──────┴────────────────────────────────────────┴───────────┐  │
│  │  profisafe_core                                           │  │
│  │  ┌─────────────┐ ┌──────────────┐ ┌────────────────────┐  │  │
│  │  │ Sequence    │ │  Watchdog    │ │  Codename (F_Par)  │  │  │
│  │  │ Monitoring  │ │  Timer       │ │  Validation        │  │  │
│  │  └──────┬──────┘ └──────┬───────┘ └────────┬───────────┘  │  │
│  │         │               │                  │              │  │
│  │  ┌──────┴───────────────┴──────────────────┴───────────┐  │  │
│  │  │              State Machine                          │  │  │
│  │  │  POWER_ON → F_PAR_EXCHANGE → DATA_EXCHANGE          │  │  │
│  │  │                          ↘ PASSIVATION ↙             │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                     PROFIsafe Protocol Layer                     │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Telegram Encode / Decode                                 │   │
│  │                                                          │   │
│  │  Status/Control ─── Safety Data ─── CRC1 ─── SeqNr ─── │   │
│  │                                      VCN ─── CRC2       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌────────────────────┐     ┌────────────────────┐              │
│  │  CRC1 Engine       │     │  CRC2 Engine       │              │
│  │  Poly 0x755B (16b) │     │  Poly 0x1D  (8b)   │              │
│  └────────────────────┘     └────────────────────┘              │
├─────────────────────────────────────────────────────────────────┤
│                     Black-Channel Transport                      │
│                                                                 │
│  ┌──────────────────────────┐  ┌─────────────────────────────┐  │
│  │ PROFINET IO (UDP/IP)     │  │ PROFIBUS DP (FDL Layer 2)   │  │
│  │ RT Class 1 / Class 3    │  │ DP-V0 / DP-V1 cyclic        │  │
│  └──────────┬───────────────┘  └──────────────┬──────────────┘  │
├─────────────┼─────────────────────────────────┼─────────────────┤
│  ┌──────────┴───────────────┐  ┌──────────────┴──────────────┐  │
│  │ 100BASE-TX Ethernet      │  │ RS-485                      │  │
│  │ (Full Duplex, 100 Mbit)  │  │ (1.5 / 12 Mbit)            │  │
│  └──────────────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

See [docs/architecture.md](docs/architecture.md) for the full data-flow description and timing analysis.

---

## Telegram Format

PROFIsafe telegrams are appended to the standard PROFINET/PROFIBUS process data.  The safety container has a fixed structure regardless of transport:

### Standard telegram (up to 12 bytes safety data)

| Offset | Length | Field | Description |
|--------|--------|-------|-------------|
| 0 | 1 | **Status/Control** | Bit field — see below |
| 1 | 1–12 | **Safety process data** | Application payload (configurable length) |
| N+1 | 2 | **CRC1** | 16-bit CRC over Status + Safety data, poly `0x755B` |
| N+3 | 2 | **Consecutive number** | Monotonic counter, wraps at 65535 |
| N+5 | 1 | **Virtual consecutive number** | Seed for CRC2 |
| N+6 | 1 | **CRC2** | 8-bit CRC over Consecutive number + VCN, poly `0x1D` |

`N` = length of safety process data.  Total telegram: `N + 7` bytes.  Minimum 8 bytes (1 byte data), maximum 123 bytes (116 bytes data, extended mode).

### Status/Control byte (F-Device → F-Host)

| Bit | Name | Description |
|-----|------|-------------|
| 7 | `Toggle_d` | Toggle bit — alternates each cycle when connection is live |
| 6 | `FV_activated` | Fail-safe values active (device is in safe state) |
| 5 | `Device_Fault` | Internal device fault detected |
| 4 | `CE_CRC` | Communication error: CRC mismatch on received telegram |
| 3 | `CE_Timeout` | Communication error: watchdog timeout |
| 2 | `CE_SeqNr` | Communication error: consecutive number out of window |
| 1 | `F_Par_CRC_ok` | F-Parameter CRC validated successfully |
| 0 | Reserved | — |

### Control byte (F-Host → F-Device)

| Bit | Name | Description |
|-----|------|-------------|
| 7 | `Toggle_h` | Toggle bit — must mirror `Toggle_d` within one cycle |
| 6 | `Activate_FV` | Request device to activate fail-safe values |
| 5 | `R` | Reserved |
| 4 | `R` | Reserved |
| 3 | `R` | Reserved |
| 2 | `R` | Reserved |
| 1 | `F_Par_Req` | Request F-Parameter exchange (during connection setup) |
| 0 | `iParOK` | Initial parameters accepted |

---

## Safety Mechanisms

This stack implements all safety measures required by IEC 61784-3-3:

### 1. CRC1 — Safety Data Integrity

- **Polynomial:** `0x755B` (CRC-16, as specified in IEC 61784-3)
- **Scope:** Status/Control byte + Safety process data
- **Seed:** Derived from F_Par_CRC (codename), making it unique per connection
- **Hamming distance:** 4 for data lengths up to 242 bytes
- **Detection:** Single, double, and odd-count bit errors; most burst errors up to 16 bits

### 2. CRC2 — Sequence Integrity

- **Polynomial:** `0x1D` (CRC-8)
- **Scope:** Consecutive number (2 bytes) + Virtual consecutive number (1 byte)
- **Purpose:** Detects insertions, deletions, and reordering of telegrams
- **Seed:** `F_Par_CRC` low byte XOR `F_Source_Add` — unique per connection pair

### 3. Consecutive Number Monitoring

- Monotonically incrementing 16-bit counter in each telegram
- Receiver checks the counter is within a configurable tolerance window (default: ±3)
- Detects: lost telegrams, repeated telegrams, delayed telegrams
- VCN (Virtual Consecutive Number) adds an extra dimension for multi-connection disambiguation

### 4. Watchdog Timeout

- Configurable per-channel timeout (default 100 ms, minimum 16 ms = 1 cycle)
- Starts when a valid telegram is last received
- Expiry triggers passivation — device goes to fail-safe values
- Timer is re-armed on every valid telegram reception

### 5. Toggle Bit Protocol

- F-Host and F-Device each maintain a toggle bit
- F-Device sets `Toggle_d`, F-Host mirrors it back as `Toggle_h`
- If the mirror does not match within one cycle, the connection is declared faulty
- Detects masquerading and unintended loopback

### 6. Codename / F-Parameter Validation

- F-Parameters (safety address, data length, watchdog time, etc.) are exchanged during connection setup
- A CRC (F_Par_CRC) is computed over all F-Parameters — this is the "codename"
- The codename is used as seed for CRC1/CRC2 — every connection has unique CRC behaviour
- Protects against: wrong device, wrong address, parameter corruption

### 7. Passivation

- Any safety violation (CRC error, timeout, sequence error, toggle mismatch) triggers immediate passivation
- Passivated channels output **fail-safe values** (typically zero / safe state)
- Recovery requires full reconnection: F-Parameter re-exchange and state machine restart
- Passivation latency: < 1 cycle (16 ms)

---

## State Machine

```
                    ┌──────────────┐
                    │   POWER_ON   │
                    │  (all clear) │
                    └──────┬───────┘
                           │ init()
                           ▼
                    ┌──────────────┐
              ┌─────│  F_PAR_REQ   │  ← F-Host sends F_Par_Req=1
              │     └──────┬───────┘
              │            │ F-Device receives valid F-Parameters
              │            ▼
              │     ┌──────────────┐
              │     │  F_PAR_ACK   │  ← F-Device validates F_Par_CRC, sends ack
              │     └──────┬───────┘
              │            │ F-Host receives F_Par_CRC_ok=1
              │            ▼
              │     ┌──────────────┐
              │     │ TOGGLE_INIT  │  ← First toggle exchange
              │     └──────┬───────┘
              │            │ Toggle_d mirrored successfully
              │            ▼
              │     ┌──────────────────┐
              │     │  DATA_EXCHANGE   │  ← Normal operation
              │     │  (RUNNING)       │  ← CRC-protected telegrams
              │     └────┬──────┬──────┘
              │          │      │
              │   timeout│      │ CRC error / SeqNr error / Toggle mismatch
              │          │      │
              │          ▼      ▼
              │     ┌──────────────┐
              └────►│ PASSIVATION  │  ← Fail-safe values active
                    │ (safe state) │
                    └──────┬───────┘
                           │ reset() — requires operator action or host command
                           ▼
                    ┌──────────────┐
                    │  F_PAR_REQ   │  (restart connection)
                    └──────────────┘
```

Full transition table with guards and actions: [docs/state-machine.md](docs/state-machine.md)

---

## Key Features

| Feature | Detail |
|---------|--------|
| Deterministic timing | 16 ms fixed cycle on ARM Cortex-M4 (XMC4500 @ 120 MHz) |
| Zero dynamic allocation | All buffers statically sized — no `malloc`, no heap |
| SIL 3 coding standard | IEC 61508-3 systematic capability, internal SIL 3 coding standard |
| Dual-channel support | Channel A / Channel B cross-monitoring for Cat 4 (PLe) |
| Black-channel principle | Safety layer independent of PROFINET / PROFIBUS transport |
| F-Host and F-Device | Both roles implemented — configurable per channel |
| Group status | Activation, blocking, device fault, channel fault evaluation |
| Configurable data length | 1–12 bytes standard, up to 123 bytes extended mode |
| Multi-channel | Up to 16 simultaneous PROFIsafe channels |

---

## Performance

Measured on Infineon XMC4500 Relax Lite Kit (ARM Cortex-M4F, 120 MHz, no cache, flash wait-states = 3):

| Metric | Value | Notes |
|--------|-------|-------|
| Cycle time | **16 ms** deterministic | SysTick-driven, jitter < 2 µs |
| CRC1 computation (12 B data) | **< 8 µs** | Table-driven, 256-entry lookup |
| CRC2 computation (3 B) | **< 2 µs** | Table-driven |
| Full telegram encode | **< 15 µs** | Status + data + CRC1 + SeqNr + CRC2 |
| Full telegram decode + validate | **< 20 µs** | CRC check + SeqNr + watchdog |
| RAM per channel | **1.2 KB** | State, counters, buffers, F-Parameters |
| Total code size (`.text`) | **8.4 KB** | All modules, `-Os` optimisation |
| Max simultaneous channels | **16** | Limited by configuration, not resources |
| Packet loss tolerance | **3** consecutive | Configurable via F-Parameters |
| Passivation latency | **< 1 cycle** | Immediate on detection, output in same cycle |
| Watchdog resolution | **1 ms** | DWT cycle counter based |

---

## Module Structure

```
src/
├── profisafe_core.c       Main API: init, process_rx, build_tx, passivate, reset
├── profisafe_core.h       Public header — channel struct, config, error codes (PUBLIC)
├── profisafe_crc.c        CRC1 (16-bit, poly 0x755B) and CRC2 (8-bit, poly 0x1D)
├── profisafe_crc.h        CRC function declarations
├── profisafe_seqmon.c     Consecutive number tracking, VCN management
├── profisafe_seqmon.h     Sequence monitor interface
├── profisafe_fpar.c       F-Parameter exchange, codename (F_Par_CRC) computation
├── profisafe_fpar.h       F-Parameter types and validation API
├── profisafe_watchdog.c   Per-channel watchdog timer with DWT backing
├── profisafe_watchdog.h   Watchdog interface
├── profisafe_state.c      Connection state machine (transitions, guards, actions)
├── profisafe_state.h      State enum, transition function
├── profisafe_diag.c       Diagnostic counters (CRC errors, timeouts, passivations)
├── profisafe_diag.h       Diagnostic readout interface
├── profisafe_types.h      Shared typedefs, constants, error codes, bit masks
└── NOTICE.md              Proprietary code notice

tests/
├── test_crc.c             CRC1/CRC2 test vectors from IEC 61784-3
├── test_seqmon.c          Sequence number window tests
├── test_state.c           State machine transition tests
├── test_integration.c     Full encode-decode round-trip with fault injection
└── NOTICE.md              Proprietary code notice

docs/
├── architecture.md        Layer model, data flow, timing budget
├── state-machine.md       Complete state transition table
├── crc-specification.md   Polynomial definitions, lookup tables, test vectors
├── integration-guide.md   How to integrate into your safety controller
└── testing-report.md      Unit test results, coverage, HW test summary
```

---

## Integration Example

```c
#include "profisafe_core.h"

/* --- Configuration (typically from F-Parameters / GSD) --- */
static const profisafe_config_t cfg = {
    .role             = PROFISAFE_ROLE_F_DEVICE,
    .safety_data_len  = 4,                  /* 4 bytes process data         */
    .watchdog_time_ms = 100,                /* 100 ms watchdog              */
    .f_source_addr    = 0x0001,             /* F-Host address               */
    .f_dest_addr      = 0x0002,             /* This device's address        */
    .seq_nr_tolerance = 3,                  /* Accept ±3 in sequence number */
};

static profisafe_channel_t ch;

void safety_init(void)
{
    profisafe_err_t err = profisafe_init(&ch, &cfg);
    /* err == PROFISAFE_OK on success */
}

/* Called every 16 ms from the SysTick handler */
void safety_cycle(const uint8_t *rx_buf, uint16_t rx_len,
                  uint8_t *tx_buf, uint16_t tx_buf_size)
{
    /* 1. Process received telegram (CRC check, SeqNr, watchdog) */
    profisafe_err_t err = profisafe_process_rx(&ch, rx_buf, rx_len);

    if (err == PROFISAFE_OK) {
        /* 2. Read validated safety input data */
        uint8_t safe_in[4];
        profisafe_get_safety_data(&ch, safe_in, sizeof(safe_in));

        /* 3. Run safety function (application-specific) */
        uint8_t safe_out[4];
        safety_function(safe_in, safe_out);

        /* 4. Write safety output data */
        profisafe_set_safety_data(&ch, safe_out, sizeof(safe_out));
    }

    /* 5. Build transmit telegram (always — even in passivation) */
    uint16_t tx_len = profisafe_build_tx(&ch, tx_buf, tx_buf_size);

    /* tx_buf now contains the PROFIsafe telegram to be sent via
     * PROFINET IO or PROFIBUS DP in the next bus cycle. */
}
```

See [docs/integration-guide.md](docs/integration-guide.md) for the full step-by-step walkthrough.

---

## Testing

| Category | Tests | Pass | Coverage |
|----------|-------|------|----------|
| CRC1 / CRC2 test vectors | 24 | 24 | 100% |
| Sequence number monitoring | 18 | 18 | 100% |
| State machine transitions | 32 | 32 | 100% |
| Watchdog timeout | 12 | 12 | 100% |
| F-Parameter exchange | 14 | 14 | 100% |
| Toggle bit protocol | 8 | 8 | 100% |
| Full round-trip integration | 22 | 22 | — |
| Fault injection (CRC flip, timeout, seq gap) | 28 | 28 | — |
| **Total** | **158** | **158** | **98.2% branch** |

Hardware integration tested on:
- Infineon XMC4500 Relax Lite Kit + Hilscher netX 90 (PROFINET)
- Infineon XMC4500 + Anybus CompactCom 40 (PROFIBUS DP)
- Wireshark + PROFIsafe dissector for packet-level validation

See [docs/testing-report.md](docs/testing-report.md) for detailed results.

---

## Build

Requires DAVE IDE 4.5 / ARM GCC toolchain:

```bash
# Build (from DAVE IDE project root)
make -C Debug all

# Run unit tests on host (x86, uses POSIX test harness)
cd tests/
make test
```

Compiler flags used in the safety build:

```
-std=c99 -Wall -Wextra -Werror -Wconversion -Wsign-conversion
-fno-builtin -ffreestanding -Os
-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16
```

---

## References

| Document | Description |
|----------|-------------|
| IEC 61784-3-3:2021 | PROFIsafe protocol specification |
| IEC 61508:2010 | Functional safety of E/E/PE systems |
| IEC 62061:2021 | Safety of machinery — functional safety of SRP/CS |
| PI Spec: PROFIsafe — Profile for Safety Technology | PROFIBUS & PROFINET International |
| PI Spec: PROFINET IO — Application Layer Protocol | Cyclic RT/IRT data exchange |
| Infineon XMC4500 Reference Manual | ARM Cortex-M4F, peripheral registers |
| internal SIL 3 coding standard | SIL 3 coding standard (internal) |

---

## Access

> **Source code in `src/` and `tests/` is proprietary.**
>
> The architecture documentation in `docs/` and this README are public to demonstrate the design approach and protocol knowledge.  For code review access or collaboration, contact:
>
> **Umang Panchal** — [GitHub](https://github.com/ichumang) · [github.com/ichumang](https://github.com/ichumang)

---

## License

Proprietary — see [NOTICE.md](NOTICE.md).
