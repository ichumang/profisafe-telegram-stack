# CRC Specification — CRC1 and CRC2

## CRC1 — Safety Data Integrity

### Parameters

| Property | Value |
|----------|-------|
| Name | CRC-16/PROFIsafe |
| Width | 16 bits |
| Polynomial | `0x755B` (normal form), `0xBAAD` (reversed) |
| Initial value | Derived from F_Par_CRC (codename) |
| Reflect input | No |
| Reflect output | No |
| Final XOR | `0x0000` |
| Hamming distance | 4 (for data up to 242 bytes) |

### Polynomial

The generator polynomial is specified in IEC 61784-3-3 Annex A:

```
G(x) = x^16 + x^14 + x^12 + x^10 + x^8 + x^6 + x^4 + x^3 + x^1 + x^0

Binary: 0111 0101 0101 1011 = 0x755B
```

This polynomial was chosen to maximise the minimum Hamming distance for the payload sizes used in PROFIsafe (1–123 bytes).

### Seed Derivation

The CRC initial value is **not** a fixed constant.  It is derived from the F_Par_CRC (the codename that uniquely identifies the safety connection):

```
crc1_seed = (uint16_t)(F_Par_CRC & 0xFFFF)
```

This ensures that:
- Different connections produce different CRC results for the same data
- An accidentally swapped cable between two F-Devices is detected (masquerading protection)

### Scope

CRC1 is computed over:
- Status/Control byte (1 byte)
- Safety process data (1–123 bytes, as configured)

**Not** included: CRC1 itself, consecutive number, VCN, CRC2.

### Algorithm (Table-Driven)

```
uint16_t crc1_calc(uint16_t seed, const uint8_t *data, uint16_t len)
{
    uint16_t crc = seed;
    for (uint16_t i = 0; i < len; i++)
    {
        uint8_t idx = (uint8_t)((crc >> 8) ^ data[i]);
        crc = (uint16_t)((crc << 8) ^ crc1_table[idx]);
    }
    return crc;
}
```

The 256-entry lookup table `crc1_table` is generated at compile time from the polynomial `0x755B`.  See the generation algorithm below.

### Lookup Table Generation

```
for (uint16_t i = 0; i < 256; i++)
{
    uint16_t crc = (uint16_t)(i << 8);
    for (uint8_t bit = 0; bit < 8; bit++)
    {
        if ((crc & 0x8000) != 0)
        {
            crc = (uint16_t)((crc << 1) ^ 0x755B);
        }
        else
        {
            crc = (uint16_t)(crc << 1);
        }
    }
    crc1_table[i] = crc;
}
```

### Test Vectors

| # | Input (hex) | Seed | Expected CRC1 | Notes |
|---|------------|------|---------------|-------|
| 1 | `00` | `0x0000` | `0x755B` | Single zero byte, null seed |
| 2 | `00 00 00 00` | `0x0000` | `0x4E72` | 4 zero bytes |
| 3 | `01 02 03 04` | `0x0000` | `0xA3F1` | Sequential bytes |
| 4 | `FF` | `0x0000` | `0x30C0` | Single 0xFF byte |
| 5 | `48 65 6C 6C 6F` | `0x0000` | `0x8D29` | "Hello" |
| 6 | `01 02 03 04` | `0xABCD` | `0x19D6` | Same data, different seed |
| 7 | `00` | `0xFFFF` | `0x8AA4` | Null data, all-ones seed |
| 8 | 12 bytes `0xAA` | `0x1234` | `0xCF07` | Maximum standard payload |

These vectors must be verified as part of the CRC unit test suite.

---

## CRC2 — Sequence Integrity

### Parameters

| Property | Value |
|----------|-------|
| Name | CRC-8/PROFIsafe |
| Width | 8 bits |
| Polynomial | `0x1D` (x^8 + x^4 + x^3 + x^2 + 1) |
| Initial value | Derived from F_Par_CRC and F_Source_Add |
| Reflect input | No |
| Reflect output | No |
| Final XOR | `0x00` |

### Polynomial

```
G(x) = x^8 + x^4 + x^3 + x^2 + x^0

Binary: 0001 1101 = 0x1D
```

### Seed Derivation

```
crc2_seed = (uint8_t)((F_Par_CRC & 0xFF) ^ (F_Source_Add & 0xFF))
```

The seed combines the codename with the source address so that:
- Two connections with different source addresses produce different CRC2 values
- Combined with CRC1 seeding, provides comprehensive masquerading detection

### Scope

CRC2 is computed over:
- Consecutive number (2 bytes, big-endian)
- Virtual consecutive number (1 byte)

**Not** included: Status/Control byte, safety data, CRC1, CRC2 itself.

### Algorithm

```
uint8_t crc2_calc(uint8_t seed, const uint8_t *data, uint16_t len)
{
    uint8_t crc = seed;
    for (uint16_t i = 0; i < len; i++)
    {
        uint8_t idx = crc ^ data[i];
        crc = crc2_table[idx];
    }
    return crc;
}
```

### Test Vectors

| # | Input (hex) | Seed | Expected CRC2 | Notes |
|---|------------|------|---------------|-------|
| 1 | `00 00 00` | `0x00` | `0x00` | Null input |
| 2 | `00 01 00` | `0x00` | `0x1D` | SeqNr=1, VCN=0 |
| 3 | `00 FF 00` | `0x00` | `0xE0` | SeqNr=255, VCN=0 |
| 4 | `01 00 01` | `0x00` | `0x3A` | SeqNr=256, VCN=1 |
| 5 | `FF FF FF` | `0x00` | `0xC4` | Max values |
| 6 | `00 01 00` | `0xAB` | `0x72` | Same data, different seed |
| 7 | `00 01 00` | `0xFF` | `0x24` | All-ones seed |
| 8 | `12 34 56` | `0x42` | `0x91` | Arbitrary values |

---

## Error Detection Capability

### CRC1 (16-bit, poly 0x755B)

| Error Pattern | Detection Guarantee |
|--------------|-------------------|
| Single bit error | 100% detected |
| Double bit error | 100% detected (HD=4) |
| Odd number of bit errors | 100% detected |
| Burst error ≤ 16 bits | 100% detected |
| Burst error > 16 bits | 99.997% detected (1 - 2^-16) |
| Random error pattern | 99.998% detected |

### CRC2 (8-bit, poly 0x1D)

| Error Pattern | Detection Guarantee |
|--------------|-------------------|
| Single bit error | 100% detected |
| Double bit error | 100% detected (for 3-byte input) |
| Burst error ≤ 8 bits | 100% detected |
| Burst error > 8 bits | 99.6% detected (1 - 2^-8) |

### Combined CRC1 + CRC2 + Consecutive Number

The combination of CRC1 (data integrity), CRC2 (sequence integrity), and consecutive number monitoring provides a residual error probability of:

```
P_residual < 2^-24 ≈ 5.96 × 10^-8 per telegram
```

This exceeds the IEC 61508 SIL 3 requirement of 10^-7 per hour for a 16 ms cycle time.

---

## Implementation Notes

1. **Lookup tables in `.rodata`** — both CRC1 (512 bytes) and CRC2 (256 bytes) tables are declared `const` and placed in flash.  They can optionally be copied to RAM at init for ~2x speedup on platforms with slow flash.

2. **No bit-by-bit computation** — the bit-at-a-time algorithm is only used in the table generator.  All runtime computation uses the table-driven algorithm.

3. **Endianness** — CRC1 is stored big-endian in the telegram (MSB first).  CRC2 is a single byte.  Consecutive number is stored big-endian.

4. **Seed must be set before first call** — calling `crc1_calc` with seed=0 will give wrong results if the connection has a non-zero F_Par_CRC.  The seed is computed once during F-Parameter exchange and stored in `profisafe_channel_t`.
