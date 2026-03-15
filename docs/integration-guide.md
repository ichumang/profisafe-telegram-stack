# Integration Guide

How to integrate the PROFIsafe telegram stack into an existing safety controller project.

## Prerequisites

- ARM Cortex-M4 or equivalent (tested on XMC4500, XMC4800, STM32F4)
- C99 compiler with `-ffreestanding` support
- Cyclic communication layer providing raw byte buffers (PROFINET IO or PROFIBUS DP)
- DWT cycle counter available for watchdog timing (or alternative microsecond timer)

## Step 1 — Add Source Files

Copy the `src/` directory into your project tree.  You need all `.c` and `.h` files:

```
your_project/
├── src/
│   ├── safety/
│   │   ├── profisafe_core.c
│   │   ├── profisafe_core.h
│   │   ├── profisafe_crc.c
│   │   ├── profisafe_crc.h
│   │   ├── profisafe_seqmon.c
│   │   ├── profisafe_seqmon.h
│   │   ├── profisafe_fpar.c
│   │   ├── profisafe_fpar.h
│   │   ├── profisafe_watchdog.c
│   │   ├── profisafe_watchdog.h
│   │   ├── profisafe_state.c
│   │   ├── profisafe_state.h
│   │   ├── profisafe_diag.c
│   │   ├── profisafe_diag.h
│   │   └── profisafe_types.h
│   └── app/
│       └── main.c
```

Add all `.c` files to your build system.  No external dependencies required — the stack uses only `<stdint.h>`, `<stddef.h>`, and `<string.h>` from the standard library.

## Step 2 — Configure the Channel

Each PROFIsafe connection is represented by a `profisafe_channel_t` instance.  Declare one per safety connection (typically one per safe I/O module):

```c
#include "profisafe_core.h"

static profisafe_channel_t safe_ch0;

static const profisafe_config_t safe_ch0_cfg = {
    .role             = PROFISAFE_ROLE_F_DEVICE,
    .safety_data_len  = 4,        /* bytes of safety process data       */
    .watchdog_time_ms = 100,      /* communication watchdog timeout     */
    .f_source_addr    = 0x0001,   /* address of the F-Host              */
    .f_dest_addr      = 0x0002,   /* our F-Device address               */
    .seq_nr_tolerance = 3,        /* consecutive number window ±3       */
    .fail_safe_values = {0},      /* output on passivation (all zeros)  */
};
```

Configuration values come from:
- **GSD file** (PROFINET) or **GSD/GSE file** (PROFIBUS): data length, addresses
- **Safety PLC project** (TIA Portal, STEP 7 F-System): watchdog time, F-Parameters
- **System design**: fail-safe values, role

## Step 3 — Initialise at Startup

Call `profisafe_init()` once during system startup, before enabling the communication stack:

```c
void safety_startup(void)
{
    profisafe_err_t err = profisafe_init(&safe_ch0, &safe_ch0_cfg);
    if (err != PROFISAFE_OK) {
        /* configuration error — check err code */
        enter_safe_state();
        return;
    }

    /* Channel is now in F_PAR_REQ state, waiting for F-Parameter exchange */
}
```

## Step 4 — Cyclic Call in the Safety Task

The stack must be called once per communication cycle (16 ms on our platform).  This is typically done in the SysTick or timer interrupt handler:

```c
/* Called from SysTick every 16 ms */
void safety_cycle(void)
{
    uint8_t rx_container[PROFISAFE_MAX_TELEGRAM_LEN];
    uint16_t rx_len;

    /* 1. Get received PROFIsafe container from transport layer */
    rx_len = profinet_get_safety_data(SLOT_1, SUBSLOT_1,
                                      rx_container, sizeof(rx_container));

    /* 2. Process the received telegram */
    profisafe_err_t err = profisafe_process_rx(&safe_ch0,
                                               rx_container, rx_len);

    /* 3. Read validated safety inputs (only valid if err == OK) */
    uint8_t safe_inputs[4] = {0};
    if (err == PROFISAFE_OK) {
        profisafe_get_safety_data(&safe_ch0, safe_inputs, sizeof(safe_inputs));
    }

    /* 4. Execute safety function */
    uint8_t safe_outputs[4];
    run_safety_function(safe_inputs, safe_outputs,
                        profisafe_get_state(&safe_ch0));

    /* 5. Write safety outputs */
    profisafe_set_safety_data(&safe_ch0, safe_outputs, sizeof(safe_outputs));

    /* 6. Build transmit telegram */
    uint8_t tx_container[PROFISAFE_MAX_TELEGRAM_LEN];
    uint16_t tx_len = profisafe_build_tx(&safe_ch0,
                                          tx_container, sizeof(tx_container));

    /* 7. Hand to transport layer for transmission */
    profinet_set_safety_data(SLOT_1, SUBSLOT_1, tx_container, tx_len);
}
```

## Step 5 — Handle State Changes

Monitor the channel state to drive your safety application:

```c
profisafe_state_t state = profisafe_get_state(&safe_ch0);

switch (state) {
    case PROFISAFE_STATE_DATA_EXCHANGE:
        /* Normal operation — safety data is valid */
        break;

    case PROFISAFE_STATE_PASSIVATION:
        /* Safety violation — outputs are fail-safe */
        /* Application must enter safe state */
        break;

    case PROFISAFE_STATE_F_PAR_REQ:
    case PROFISAFE_STATE_F_PAR_ACK:
    case PROFISAFE_STATE_TOGGLE_INIT:
        /* Connection establishing — outputs are fail-safe */
        break;

    default:
        break;
}
```

## Step 6 — Diagnostics (Optional)

Read diagnostic counters for commissioning and trouble-shooting:

```c
profisafe_diag_t diag;
profisafe_get_diagnostics(&safe_ch0, &diag);

/* Available counters:
 *   diag.crc1_errors     — CRC1 mismatches
 *   diag.crc2_errors     — CRC2 mismatches
 *   diag.seqnr_errors    — Consecutive number out-of-window
 *   diag.timeouts        — Watchdog timeouts
 *   diag.toggle_errors   — Toggle bit mismatches
 *   diag.passivations    — Total passivation count
 *   diag.rx_telegrams    — Total valid received telegrams
 *   diag.tx_telegrams    — Total transmitted telegrams
 */
```

## Transport Layer Adapter

The stack itself is transport-agnostic.  You need a thin adapter that extracts the PROFIsafe container from the PROFINET IO or PROFIBUS DP cyclic frame.

### PROFINET IO adapter (example)

The PROFIsafe container is located within the IOCR data at the submodule's data offset.  On Hilscher netX:

```c
uint16_t profinet_get_safety_data(uint16_t slot, uint16_t subslot,
                                   uint8_t *buf, uint16_t buf_size)
{
    /* Read from DPM (dual-port memory) at the submodule's IO offset */
    uint16_t offset = get_submodule_data_offset(slot, subslot);
    uint16_t len    = get_submodule_data_length(slot, subslot);

    if (len > buf_size) { len = buf_size; }
    memcpy(buf, &dpm_input_area[offset], len);
    return len;
}
```

### PROFIBUS DP adapter (example)

On Anybus CompactCom:

```c
uint16_t profibus_get_safety_data(uint8_t slot,
                                   uint8_t *buf, uint16_t buf_size)
{
    uint16_t len = anybus_get_process_data_length(slot);
    if (len > buf_size) { len = buf_size; }
    anybus_read_process_data(slot, buf, len);
    return len;
}
```

## Compiler Settings

For SIL 3 compliance, the following compiler flags are mandatory:

```
-std=c99 -Wall -Wextra -Werror -Wconversion -Wsign-conversion
-fno-builtin -ffreestanding -Os
```

Platform-specific:

```
# XMC4500
-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16

# STM32F4
-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16 -DSTM32F407xx
```

Do **not** use:
- `-fno-strict-aliasing` (the stack is aliasing-safe)
- `-ffast-math` (unsafe floating-point optimisations)
- Any compiler extension pragmas or builtins

## Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Immediate passivation after F_Par_ACK | F_Par_CRC mismatch between host and device | Verify GSD file parameters match PLC project |
| CRC1 errors every ~1000 telegrams | EMC interference on bus cable | Check shielding, termination, cable routing |
| Watchdog timeout at startup | Communication cycle not started before watchdog expires | Call `profisafe_init()` after transport layer is running |
| Toggle bit mismatch | Host and device cycle times not synchronised | Ensure both run at the same cycle time (±10%) |
| SeqNr error after power glitch | SeqNr counter not re-initialised after transport reconnect | Call `profisafe_reset()` when transport layer reconnects |
