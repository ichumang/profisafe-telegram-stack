# Testing Report

## Summary

| Category | Tests | Passed | Failed | Skip | Branch Coverage |
|----------|-------|--------|--------|------|----------------|
| CRC1 / CRC2 | 24 | 24 | 0 | 0 | 100% |
| Sequence monitoring | 18 | 18 | 0 | 0 | 100% |
| State machine | 32 | 32 | 0 | 0 | 100% |
| Watchdog timer | 12 | 12 | 0 | 0 | 100% |
| F-Parameter exchange | 14 | 14 | 0 | 0 | 100% |
| Toggle bit protocol | 8 | 8 | 0 | 0 | 100% |
| Full integration (encode/decode) | 22 | 22 | 0 | 0 | — |
| Fault injection | 28 | 28 | 0 | 0 | — |
| **Total** | **158** | **158** | **0** | **0** | **98.2%** |

Remaining 1.8% uncovered branches: defensive `default:` cases in switch statements that cannot be reached under normal or injected fault conditions.  These are required by internal SIL 3 coding standard 1 (SESE / defensive coding).

## Test Environment

### Host-based unit tests (x86)

- **OS:** Ubuntu 22.04 LTS (CI runner)
- **Compiler:** GCC 12.3, `-std=c99 -Wall -Wextra -Werror -O0 -g --coverage`
- **Framework:** Custom minimal test harness (assert macros, no external dependency)
- **Coverage:** `gcov` + `lcov`, report generated per module

### Target-based integration tests

- **Hardware:** Infineon XMC4500 Relax Lite Kit (ARM Cortex-M4F, 120 MHz)
- **PROFINET:** Hilscher netX 90 evaluation board (PROFINET IO Device)
- **PROFIBUS:** Anybus CompactCom 40 (PROFIBUS DP Slave)
- **Controller:** Siemens S7-1516F (F-Host, TIA Portal V18)
- **Capture:** Wireshark 4.2 with PROFIsafe dissector plugin

---

## Unit Test Details

### CRC1 Tests (24 tests)

| Test | Description | Result |
|------|-------------|--------|
| `test_crc1_null_seed_null_data` | Seed=0, data=0x00 → expected 0x755B | PASS |
| `test_crc1_null_seed_4_zeros` | Seed=0, 4×0x00 → expected 0x4E72 | PASS |
| `test_crc1_null_seed_sequential` | Seed=0, [01 02 03 04] → expected 0xA3F1 | PASS |
| `test_crc1_null_seed_ff` | Seed=0, 0xFF → expected 0x30C0 | PASS |
| `test_crc1_null_seed_hello` | Seed=0, "Hello" → expected 0x8D29 | PASS |
| `test_crc1_seeded` | Seed=0xABCD, [01 02 03 04] → expected 0x19D6 | PASS |
| `test_crc1_all_ones_seed` | Seed=0xFFFF, 0x00 → expected 0x8AA4 | PASS |
| `test_crc1_max_payload` | Seed=0x1234, 12×0xAA → expected 0xCF07 | PASS |
| `test_crc1_single_bit_flip_detected` | Flip bit 3 of byte 2 → CRC differs | PASS |
| `test_crc1_double_bit_flip_detected` | Flip 2 random bits → CRC differs | PASS |
| `test_crc1_burst_error_16bit` | 16-bit burst in middle → detected | PASS |
| `test_crc1_all_zeros_vs_all_ones` | Different data → different CRC | PASS |
| ... (12 more polynomial/boundary tests) | | ALL PASS |

### Sequence Monitor Tests (18 tests)

| Test | Description | Result |
|------|-------------|--------|
| `test_seqmon_nominal_increment` | SeqNr +1 each cycle → accepted | PASS |
| `test_seqmon_gap_within_tolerance` | SeqNr +2 (1 lost telegram) → accepted | PASS |
| `test_seqmon_gap_at_tolerance_limit` | SeqNr +3 (tolerance=3) → accepted | PASS |
| `test_seqmon_gap_exceeds_tolerance` | SeqNr +4 (tolerance=3) → rejected | PASS |
| `test_seqmon_duplicate_rejected` | Same SeqNr twice → rejected | PASS |
| `test_seqmon_old_telegram_rejected` | SeqNr -1 → rejected | PASS |
| `test_seqmon_wraparound_65535_to_0` | 0xFFFF → 0x0000 → accepted | PASS |
| `test_seqmon_wraparound_with_gap` | 0xFFFE → 0x0001 → accepted (gap=3) | PASS |
| `test_seqmon_vcn_mismatch` | Wrong VCN → CRC2 check fails | PASS |
| ... (9 more window/edge tests) | | ALL PASS |

### State Machine Tests (32 tests)

| Test | Description | Result |
|------|-------------|--------|
| `test_state_init_to_fpar_req` | After init → F_PAR_REQ | PASS |
| `test_state_fpar_req_to_fpar_ack` | Valid F-Params received → F_PAR_ACK | PASS |
| `test_state_fpar_ack_to_toggle_init` | F_Par_CRC_ok=1 → TOGGLE_INIT | PASS |
| `test_state_toggle_init_to_data_exchange` | Toggle mirrored → DATA_EXCHANGE | PASS |
| `test_state_data_exchange_stays` | Valid telegram → stays in DATA_EXCHANGE | PASS |
| `test_state_crc_error_passivates` | CRC1 fail → PASSIVATION | PASS |
| `test_state_timeout_passivates` | Watchdog expiry → PASSIVATION | PASS |
| `test_state_seqnr_error_passivates` | SeqNr out of window → PASSIVATION | PASS |
| `test_state_toggle_mismatch_passivates` | Toggle_h != Toggle_d → PASSIVATION | PASS |
| `test_state_passivation_outputs_failsafe` | Passivated → TX contains fail-safe values | PASS |
| `test_state_reset_returns_to_fpar_req` | reset() from PASSIVATION → F_PAR_REQ | PASS |
| `test_state_fpar_timeout_passivates` | No F-Params within timeout → PASSIVATION | PASS |
| ... (20 more transition/guard tests) | | ALL PASS |

### Fault Injection Tests (28 tests)

| Test | Description | Result |
|------|-------------|--------|
| `test_inject_single_bit_crc1` | Flip 1 bit in safety data → passivation | PASS |
| `test_inject_multi_bit_crc1` | Flip 3 bits in safety data → passivation | PASS |
| `test_inject_crc2_corruption` | Corrupt CRC2 byte → passivation | PASS |
| `test_inject_seqnr_replay` | Replay old telegram → passivation | PASS |
| `test_inject_seqnr_skip_10` | Skip 10 consecutive numbers → passivation | PASS |
| `test_inject_toggle_stuck` | Toggle bit doesn't alternate → passivation | PASS |
| `test_inject_wrong_codename` | Different F_Par_CRC seed → CRC1 fails | PASS |
| `test_inject_telegram_truncation` | Short telegram → detected and passivated | PASS |
| `test_inject_telegram_extension` | Extra bytes appended → ignored (length check) | PASS |
| `test_inject_all_zeros_telegram` | 0x00 fill → CRC fails | PASS |
| `test_inject_all_ones_telegram` | 0xFF fill → CRC fails | PASS |
| `test_inject_delayed_3_cycles` | 3 missed then valid → accepted (tolerance=3) | PASS |
| `test_inject_delayed_4_cycles` | 4 missed then valid → rejected (tolerance=3) | PASS |
| `test_inject_watchdog_edge` | Telegram arrives at watchdog_time_ms - 1 → accepted | PASS |
| `test_inject_watchdog_expired` | Telegram arrives at watchdog_time_ms + 1 → passivation | PASS |
| ... (13 more injection scenarios) | | ALL PASS |

---

## Hardware Integration Test Results

### PROFINET IO (Hilscher netX 90)

| Test | Duration | Result |
|------|----------|--------|
| Connection setup (F-Param + Toggle) | 3 cycles (48 ms) | PASS |
| Sustained 1-hour data exchange | 225,000 telegrams | PASS — 0 errors |
| Cable disconnect / reconnect | 5 cycles | PASS — passivation in 1 cycle, recovery in 3 cycles |
| EMC burst (IEC 61000-4-4, Level 3) | 10 minutes | PASS — 2 CRC errors detected and passivated |
| Power supply dip 6.5V / 500ms | — | PASS — passivation + auto-recovery |

### PROFIBUS DP (Anybus CompactCom 40)

| Test | Duration | Result |
|------|----------|--------|
| Connection setup | 4 cycles (64 ms) | PASS |
| Sustained 1-hour data exchange | 225,000 telegrams | PASS — 0 errors |
| Bus termination removed | — | PASS — CRC errors detected within 2 cycles |
| Baud rate 1.5 Mbit/s | 30 minutes | PASS |
| Baud rate 12 Mbit/s | 30 minutes | PASS |

### Wireshark Packet Validation

Captured 10,000 consecutive PROFIsafe telegrams and verified:
- CRC1 correct in 100% of captured frames
- CRC2 correct in 100% of captured frames
- Consecutive number monotonically increasing (no gaps, no duplicates)
- Toggle bit alternating every cycle
- Status byte consistent with connection state

Wireshark dissector plugin: custom Lua dissector parsing PROFIsafe container within PROFINET RT frames.
