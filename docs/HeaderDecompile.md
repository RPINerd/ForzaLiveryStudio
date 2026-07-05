# Forza Horizon Header File Decompilation

This document details the binary structure of the `header` file in Forza Horizon Vinyl Groups, based on byte-level comparisons of files from multiple creators and published/unpublished states.

## File Overview

The `header` file stores metadata about a vinyl group: its name, creator, timestamps, description (when published), and a unique GUID.

Two structural variants exist:

| Variant | Size Range | Description |
|---------|-----------|-------------|
| **Draft** (unpublished) | ~145 bytes | Simple structure with null padding and trailing metadata |
| **Published** | ~133 bytes | Includes description field, published flag, duplicate GUID |

---

## Section 1: File Preamble

Fixed header that identifies the file format and contains the vinyl group name.

```
Offset  Size  Field              Example (Fr4g3z "Test")
------  ----  -----------------  ---------------------------
0x00    4     Format Version     07 00 00 00  (always 7)
0x04    4     Name Length        04 00 00 00  (chars, UTF-16LE)
0x08    N*2   Vinyl Group Name   54 00 65 00 73 00 74 00  ("Test")
```

After the name string, the next field depends on whether the vinyl is published:

### Draft (unpublished)
```
0x10  4  Null Padding  00 00 00 00
```

### Published
```
0x10  4  Description Length  06 00 00 00  (= 6 chars)
0x14  N*2 Description Text   71 00 77 00 65 00 72 00 74 00 79 00  ("qwerty")
```

---

## Section 2: Metadata Block

This section follows immediately after the vinyl name (preamble). Its absolute offset varies based on name length and description presence.

For **Fr4g3z ("Test", draft)**: starts at `0x14`
For **Fr4g3zPublished ("Test", with description)**: starts at `0x20`

### Timestamp Group (8 bytes)

```
Offset  Size  Field     Example               Notes
------  ----  ------    -------------------   ---------------------------
+0x00   2     Year      ea 07  (= 2026)       Little-endian uint16
+0x02   1     Month     06                     June
+0x03   1     Day       00                     (always 0 in observed files)
+0x04   2     Field A   02 00  (= 2)           Always 2
+0x06   2     Field B   02 00  (= 2)           Always 2
```

`ea 07` = `0x07EA` = 2026. The day field being always `00` may indicate this is not a calendar day but a flag or unused field.

### Creator-Varying Fields (10 bytes)

These fields differ between individual vinyl groups even when created by the same person.

```
Offset  Size  Field       Example (draft)     Notes
------  ----  ---------   ------------------- ---------------------------
+0x08   2     field_x     17 00  (= 23)       Varies per vinyl group
+0x0A   2     field_y     01 00  (= 1)        Varies per vinyl group
+0x0C   2     field_z     13 00  (= 19)       Varies per vinyl group
+0x0E   2     field_w_lo  b9 01  (= 441)      Lower uint16, varies
+0x10   2     field_w_hi  02 00  (= 2)        Always 2
+0x12   2     pad         00 00               Always zeros
```

The exact meaning of `field_x`, `field_y`, `field_z`, `field_w_lo` is unknown. They differ between vinyl groups regardless of creator. They may encode:
- Vinyl group type or category ID
- Number of layers or shape types
- Some form of content hash or checksum

### Creator Identity Block (12 bytes + variable name)

```
Offset  Size  Field            Example (Fr4g3z)     Notes
------  ----  ---------------  -------------------  ---------------------------
+0x14   4     creator_tag1     32 c7 6c 1c          Unique per creator
+0x18   2     creator_tag2     fa 01                Unique per creator
+0x1A   2     sep_0900         09 00  (= 9)         Constant separator
+0x1C   4     creator_len      06 00 00 00  (= 6)   Name length in chars
+0x20   N*2   creator_name     46 00 72 00 ...      UTF-16LE string
```

#### Creator Tags Observed

| Creator | tag1 | tag2 |
|---------|------|------|
| Fr4g3z | `32 c7 6c 1c` | `fa 01` |
| BTM7349 | `c8 c8 7c 52` | `f7 01` |
| SageCloth5288 | `51 d8 a8 6c` | `f1 01` |

The tag2 byte values (`fa`, `f7`, `f1`) may encode the creator name length or be part of a hash. These tags are consistent across all vinyl groups by the same creator, suggesting they are derived from the creator's Forza account ID or gamertag.

---

## Section 3: After Creator Name

### Draft (unpublished)

```
Offset          Content                           Size
------          -------                           ----
after creator   00 00 00 00 ...                   28 bytes zero padding
                01 02 00 00 00 00 00 00 00 XX     13 bytes section header
                XX 00 00 00                       (XX varies: 02, 04, 06)
                16-byte GUID                      Unique identifier
                24-byte trailing data             Metadata (6 × uint32)
```

### Published

```
Offset          Content                           Size
------          -------                           ----
after creator   01 00 00 00  (= 1)                4 bytes published flag
                00 00 00 00 00 00 00 00           8 bytes zero padding
                16-byte GUID                      First copy (no header)
                01 02 00 00 00 00 00 00 00 02     13 bytes section header
                00 00 00
                16-byte GUID                      Second copy (same GUID)
                (no trailing metadata)
```

#### Published Flag

`01 00 00 00` (`uint32 = 1`) appears immediately after the creator name in published files. Drafts have `28 bytes` of zeros instead. This flag signals that the vinyl group has been published to the Forza community.

#### Section 3 Header

```
Offset  Size  Content
------  ----  -------------------
0x00    2     01 02  (marker)
0x02    7     00 00 00 00 00 00 00  (zeros)
0x09    1     XX     (type byte: 02, 04, 06, 78...)
0x0A    3     00 00 00  (zeros)
```

The type byte at offset `+0x09` varies per vinyl group. Observed values: `02` ("Test"), `04` ("4Triangle"), `06` ("6CircleBlackMoved"), `78` (Yosip1 "2096"). It may encode:
- Layer count
- A sub-type or category identifier

#### GUID (16 bytes)

A unique 128-bit identifier formatted as a standard UUID. Format (when viewed as hex):

```
60 7D 22 9B-8D D8-2D 44-B7 CA-9A CC 79 B9 78 54
```

In published files, the GUID appears **twice**: once without a section header, and once within a section 3 header block. Both copies are identical.

#### Trailing Data (draft only)

24 bytes following the GUID in draft files. Contains up to 6 `uint32` values:

| Relative Offset | Draft Fr4g3z | Draft 6Circle | Notes |
|---------------|-------------|---------------|-------|
| +0 (u32) | 0 | 0 | Always 0 |
| +4 (u32) | 2 | 0 | Varies |
| +8 (u32) | 0 | 0 | Always 0 |
| +12 (u32) | 0 | 0 | Always 0 |
| +16 (u32) | 0 | 0 | Always 0 |
| +20 (u32) | 0 | 0 | Always 0 |

This trailing data is **absent** in published files, replaced by the duplicate GUID structure.

---

## Creator Comparison Summary

### Fr4g3z vs BTM7349 (both "Test", both draft)

| Offset | Field | Fr4g3z | BTM7349 | Same? |
|--------|-------|--------|---------|-------|
| `0x14` | timestamp | `ea 07 06 00` | `ea 07 06 00` | ✓ |
| `0x1c` | field_x | `17` (=23) | `17` (=23) | ✓ |
| `0x1e` | field_y | `01` (=1) | `04` (=4) | ✗ |
| `0x20` | field_z | `13` (=19) | `29` (=41) | ✗ |
| `0x22` | field_w_lo | `b9` (=441) | `ee` (=494) | ✗ |
| `0x28` | creator_tag1 | `32 c7 6c 1c` | `c8 c8 7c 52` | ✗ |
| `0x2c` | creator_tag2 | `fa 01` | `f7 01` | ✗ |
| `0x30` | creator_len | 6 | 7 | ✗ |
| `0x34` | creator_name | "Fr4g3z" | "BTM7349" | ✗ |
| sec3 | GUID | `607d229b...` | `d1ececc5...` | ✗ |

BTM7349 is 147 bytes vs Fr4g3z 145 bytes because "BTM7349" (7 chars = 14 bytes) is 2 bytes longer than "Fr4g3z" (6 chars = 12 bytes).

### Fr4g3z Draft vs Fr4g3zPublished (same creator, same name)

| Feature | Draft (145 bytes) | Published (133 bytes) |
|---------|-------------------|----------------------|
| After name | `00 00 00 00` (null) | `06 00 00 00` desc_len + "qwerty" |
| After creator | 28 bytes zeros | `01 00 00 00` flag + 8 bytes zeros |
| GUID count | 1 | 2 (same GUID, duplicated) |
| Trailing data | 24 bytes (6 × u32) | Removed |
| C_group | Identical | Identical |

---

## Keywords

Keywords ("Abstract", "Big", etc.) are **not stored** in any local file (`header`, `C_group`, `thumb.webp`). They exist only in Forza's server-side database, associated with the vinyl group's GUID.

---

## Summary Byte Layout

### Draft (unpublished)
```
0x00  [FormatVer=7][NameLen][VinylName UTF-16LE...]
      [00 00 00 00]  <- null pad
      [ea 07 06 00]  <- timestamp
      [02 00][02 00] <- always 2
      [field_x][field_y][field_z]
      [field_w_lo][02 00] <- field_w_hi always 2
      [00 00]            <- pad
      [creator_tag1 4B][creator_tag2 2B]
      [09 00]            <- sep constant
      [creator_len][creator_name UTF-16LE...]
      [00 * 28]          <- null padding
      [01 02][00*7][XX][00*3]  <- sec3 header (13B)
      [GUID 16B]
      [trailing 24B]
```

### Published
```
0x00  [FormatVer=7][NameLen][VinylName UTF-16LE...]
      [desc_len][desc_text UTF-16LE...]
      [ea 07 06 00]  <- timestamp
      [02 00][02 00]
      [field_x][field_y][field_z]
      [field_w_lo][02 00]
      [00 00]
      [creator_tag1 4B][creator_tag2 2B]
      [09 00]
      [creator_len][creator_name UTF-16LE...]
      [01 00 00 00]       <- published flag
      [00 * 8]            <- reduced padding
      [GUID 16B]          <- first copy, no header
      [01 02][00*7][02][00*3]  <- sec3 header
      [GUID 16B]          <- second copy, same GUID
```
