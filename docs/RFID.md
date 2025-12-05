# Snapmaker U1 RFID Filament Tag Protocol

This document describes the proprietary RFID filament tag format used by Snapmaker U1 3D printers for automatic filament detection and parameter loading.

---

## Table of Contents

1. [Overview](#overview)
2. [Hardware](#hardware)
3. [Tag Format](#tag-format)
4. [Memory Layout](#memory-layout)
5. [Data Fields](#data-fields)
6. [RSA Signature Verification](#rsa-signature-verification)
7. [RSA Public Keys](#rsa-public-keys)

---

## Overview

The Snapmaker U1 uses MIFARE Classic 1K (M1) RFID tags on filament spools to automatically identify filament properties. The system reads filament parameters including material type, color, temperature settings, and verifies authenticity using RSA-2048 digital signatures.

**Tag Specifications**:

- **Card Type**: MIFARE Classic 1K (M1_1K)
- **Total Size**: 1024 bytes
- **Protocol**: Custom Snapmaker M1 protocol
- **Security**: RSA-2048 PKCS#1 v1.5 with SHA-256
- **Signature Storage**: Sections 10-15 (288 bytes total, 256-byte signature)

---

## Hardware

### FM175xx NFC Reader

The printer uses **FM175xx** NFC reader chips to communicate with RFID tags:

**SoC-Connected Reader** (Host MCU):
- **Interface**: SPI Bus 2, Device 0
- **Speed**: 500 kHz max
- **Reset Pin**: gpiochip1/gpio25
- **Channels**:
  - Channel 1: Extruder 3 (e3)
  - Channel 2: Extruder 2 (e2)

**Extra Reader** (Host MCU):
- **Interface**: SPI Bus 2, Device 1
- **Speed**: 500 kHz max
- **Reset Pin**: gpiochip1/gpio28
- **Channels**:
  - Channel 1: Extruder 0 (e0)
  - Channel 2: Extruder 1 (e1)

**Protocol**: ISO14443A compatible (13.56 MHz)

---

## Tag Format

### Card Structure

The M1 card is organized into 16 sections of 64 bytes each (4 blocks × 16 bytes per section).

```
Section  | Blocks | Byte Range | Purpose
---------|--------|------------|---------------------------
0        | 0-3    | 0-63       | UID + Vendor + Manufacturer
1        | 4-7    | 64-127     | Version, Type, Color, SKU
2        | 8-11   | 128-191    | Physical properties + Temps
3-9      | 12-39  | 192-639    | Reserved/Unused
10-15    | 40-63  | 640-1023   | RSA-2048 Signature (288 bytes)
```

**Position Calculation**: `byte_position = (section_num × 64) + (block_num × 16) + byte_num`

---

## Memory Layout

### Section 0: Identification (Bytes 0-63)

| Offset | Length | Field | Format | Description |
|--------|--------|-------|--------|-------------|
| 0 | 4 | UID | Binary | Card unique identifier |
| 16 | 16 | VENDOR | ASCII | Brand owner/vendor name (null-terminated) |
| 32 | 16 | MANUFACTURER | ASCII | Manufacturer name (null-terminated) |

### Section 1: Material & Color (Bytes 64-127)

| Offset | Length | Field | Format | Description |
|--------|--------|-------|--------|-------------|
| 64 | 2 | VERSION | uint16_le | Protocol version |
| 66 | 2 | MAIN_TYPE | uint16_le | Material type (1=PLA, 2=PETG, 3=ABS, 4=TPU, 5=PVA) |
| 68 | 2 | SUB_TYPE | uint16_le | Sub-type (1=Basic, 2=Matte, 3=SnapSpeed) |
| 70 | 2 | TRAY | uint16_le | Tray identifier |
| 72 | 1 | COLOR_NUMS | uint8 | Number of colors (1-5) |
| 73 | 1 | ALPHA | uint8 | Alpha channel (inverted: `0xFF - value`) |
| 80 | 3 | RGB_1 | RGB24 | Primary color (R,G,B bytes) |
| 83 | 3 | RGB_2 | RGB24 | Secondary color |
| 86 | 3 | RGB_3 | RGB24 | Tertiary color |
| 89 | 3 | RGB_4 | RGB24 | Quaternary color |
| 92 | 3 | RGB_5 | RGB24 | Quinary color |
| 96 | 4 | SKU | uint32_le | Product SKU number |

### Section 2: Physical & Thermal (Bytes 128-191)

| Offset | Length | Field | Format | Unit | Description |
|--------|--------|-------|--------|------|-------------|
| 128 | 2 | DIAMETER | uint16_le | 0.01mm | Filament diameter (175 = 1.75mm) |
| 130 | 2 | WEIGHT | uint16_le | grams | Spool weight |
| 132 | 2 | LENGTH | uint16_le | meters | Filament length |
| 144 | 2 | DRYING_TEMP | uint16_le | °C | Recommended drying temperature |
| 146 | 2 | DRYING_TIME | uint16_le | hours | Recommended drying time |
| 148 | 2 | HOTEND_MAX_TEMP | uint16_le | °C | Maximum nozzle temperature |
| 150 | 2 | HOTEND_MIN_TEMP | uint16_le | °C | Minimum nozzle temperature |
| 152 | 2 | BED_TYPE | uint16_le | - | Bed surface type identifier |
| 154 | 2 | BED_TEMP | uint16_le | °C | Bed temperature |
| 156 | 2 | FIRST_LAYER_TEMP | uint16_le | °C | First layer nozzle temperature |
| 158 | 2 | OTHER_LAYER_TEMP | uint16_le | °C | Subsequent layers nozzle temperature |
| 160 | 8 | MF_DATE | ASCII | YYYYMMDD | Manufacturing date (e.g., "20241205") |
| 168 | 2 | RSA_KEY_VERSION | uint16_le | - | RSA public key version (0-9) |

### Sections 3-9: Reserved (Bytes 192-639)

Reserved for future use or padding.

### Sections 10-15: Digital Signature (Bytes 640-1023)

| Offset | Length | Purpose |
|--------|--------|---------|
| 640-927 | 288 | RSA signature storage (6 sections × 48 bytes) |

**Signature Extraction**:

- Read 48 bytes from each of sections 10-15
- Concatenate to form 288-byte buffer
- First 256 bytes contain the RSA-2048 signature
- Remaining 32 bytes are padding

---

## Data Fields

### Material Types

| Value | Name | Description |
|-------|------|-------------|
| 0 | Reserved | Reserved/Unknown |
| 1 | PLA | Polylactic Acid |
| 2 | PETG | Polyethylene Terephthalate Glycol |
| 3 | ABS | Acrylonitrile Butadiene Styrene |
| 4 | TPU | Thermoplastic Polyurethane (flexible) |
| 5 | PVA | Polyvinyl Alcohol (support material) |

### Sub-Types

| Value | Name | Description |
|-------|------|-------------|
| 0 | Reserved | Reserved/Unknown |
| 1 | Basic | Standard filament |
| 2 | Matte | Matte finish filament |
| 3 | SnapSpeed | High-speed printing optimized |

### Color Storage

Up to **5 colors** can be stored per tag:
- Each color: 3 bytes (R, G, B)
- Alpha channel: 1 byte (shared across all colors, inverted)
- Color format: `0xRRGGBB` (24-bit RGB)

**Example**:
- RGB_1 = `0xFF5500` (orange)
- ALPHA = `0xFF - 0x00 = 0xFF` (fully opaque)
- ARGB_COLOR = `0xFF` << 24 | `0xFF5500` = `0xFFFF5500`

---

## RSA Signature Verification

### Signature Process

1. **Data to Sign**: Bytes 0-639 (first 10 sections)
2. **Signature Algorithm**: RSA-2048 with PKCS#1 v1.5 padding
3. **Hash Function**: SHA-256
4. **Signature Storage**: Sections 10-15 (bytes 640-927)
5. **Key Selection**: Based on RSA_KEY_VERSION field (bytes 168-169)

### Verification Steps

```python
1. Read RSA_KEY_VERSION from bytes 168-169 (little-endian uint16)
2. Select corresponding public key (KEY_0 through KEY_9)
3. Extract signature from sections 10-15:
   - For i in range(6):
       signature_bytes += data[640 + i*64 : 640 + i*64 + 48]
   - Use first 256 bytes as signature
4. Verify: RSA_VERIFY(public_key, data[0:640], signature[0:256])
5. If verification passes: tag is authentic (OFFICIAL = True)
```

### Error Codes

| Code | Name | Description |
|------|------|-------------|
| 0 | FILAMENT_PROTO_OK | Successful parse and verification |
| -1 | FILAMENT_PROTO_ERR | General error |
| -2 | FILAMENT_PROTO_PARAMETER_ERR | Invalid parameters (wrong size, null data) |
| -3 | FILAMENT_PROTO_RSA_KEY_VER_ERR | Invalid RSA key version (not 0-9) |
| -4 | FILAMENT_PROTO_SIGN_CHECK_ERR | Signature verification failed |

---

## RSA Public Keys

Snapmaker uses **10 RSA-2048 public keys** (versions 0-9) for signature verification. Tags specify which key to use via the RSA_KEY_VERSION field.

---

**Key Information**:
- **Algorithm**: RSA-2048
- **Public Exponent**: 65537 (0x010001)
- **Format**: PKCS#1 RSA Public Key
- **Purpose**: Verify authenticity of Snapmaker official filament spools

---

*Document generated from Klipper source code analysis*
*Last updated: 2025-12-05*
