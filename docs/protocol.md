# ANT Position Protocol

## 1. Purpose

This document defines the application-level protocol used to transfer position
information between a Garmin fēnix watch and an ESP32-based radio node over ANT.

The protocol is designed for:

- compact position reporting;
- low communication overhead;
- approximately 5 m maximum coordinate quantization at the equator;
- conversion to MGRS for navigation and display;
- simple implementation on both Garmin Connect IQ and ESP32;
- future extension without changing the basic position representation.

This document initially specifies only the communication path:

```text
Garmin fēnix ──ANT──► ESP32 radio node
```

LoRa transport and communication from the radio node back to the watch are
outside the scope of this initial specification.

---

## 2. Design requirements

### 2.1 Position

The protocol shall transmit:

- latitude;
- longitude;
- position validity;
- age of the GPS position;
- sequence number.

The target coordinate quantization is approximately 5 m or better.

The intended navigation display format is primarily MGRS with 8-digit
precision:

```text
17U PV 1234 5678
```

An 8-digit MGRS coordinate represents a 10 m grid resolution.

The radio protocol does not transmit MGRS directly. Geographic coordinates
are transmitted instead and may subsequently be converted to MGRS.

### 2.2 ANT payload

A standard ANT data message contains an 8-byte application payload:

```text
8 bytes = 64 bits
```

The complete position report shall fit inside one ANT payload.

---

## 3. Position encoding

Latitude and longitude are represented as unsigned quantized integers covering
their complete geographic ranges.

### 3.1 Latitude

Latitude range:

```text
-90° <= latitude <= +90°
```

Latitude is encoded using 22 bits.

Number of possible values:

```text
2^22 = 4,194,304
```

Encoding:

```text
lat_q = round(
    ((latitude + 90) / 180) * (2^22 - 1)
)
```

where:

```text
0 <= lat_q <= 4,194,303
```

Decoding:

```text
latitude =
    (lat_q / (2^22 - 1)) * 180 - 90
```

The angular quantization step is approximately:

```text
180 / (2^22 - 1)
≈ 0.0000429°
```

This corresponds to approximately 4.8 m in latitude.

---

### 3.2 Longitude

Longitude range:

```text
-180° <= longitude <= +180°
```

Longitude is encoded using 23 bits.

Number of possible values:

```text
2^23 = 8,388,608
```

Encoding:

```text
lon_q = round(
    ((longitude + 180) / 360) * (2^23 - 1)
)
```

where:

```text
0 <= lon_q <= 8,388,607
```

Decoding:

```text
longitude =
    (lon_q / (2^23 - 1)) * 360 - 180
```

The angular quantization step is approximately:

```text
360 / (2^23 - 1)
≈ 0.0000429°
```

At the equator, this corresponds to approximately 4.8 m.

The physical east-west distance represented by one longitude step decreases
with latitude.

---

## 4. Position payload

The initial position message uses the complete 64-bit ANT payload.

### 4.1 Bit allocation

| Field | Size | Description |
|---|---:|---|
| Latitude | 22 bits | Quantized geographic latitude |
| Longitude | 23 bits | Quantized geographic longitude |
| Sequence | 4 bits | Position sequence counter |
| Age | 8 bits | Age of the position in seconds |
| GPS status | 2 bits | Position validity/status |
| Reserved | 5 bits | Reserved for future use |
| **Total** | **64 bits** | |

Conceptual representation:

```text
63                                                    0
┌─────┬──────┬────────┬─────┬────────────┬─────────────┐
│ RSV │ GPS  │  AGE   │ SEQ │ LONGITUDE  │  LATITUDE   │
│ 5 b │ 2 b  │  8 b   │ 4 b │   23 b     │    22 b     │
└─────┴──────┴────────┴─────┴────────────┴─────────────┘
```

Exact bit ranges:

| Bits | Field |
|---|---|
| 0–21 | Latitude |
| 22–44 | Longitude |
| 45–48 | Sequence |
| 49–56 | Position age |
| 57–58 | GPS status |
| 59–63 | Reserved |

Bit 0 is the least significant bit of the 64-bit payload representation.

---

## 5. Sequence counter

The sequence field contains a 4-bit unsigned counter:

```text
0–15
```

The counter increments whenever a new position report is generated.

After 15:

```text
15 → 0
```

The sequence counter allows the receiving node to distinguish a new position
from a repeated transmission of the same position report.

A change in sequence number does not necessarily imply that the geographic
coordinates have changed.

---

## 6. Position age

The age field is an 8-bit unsigned integer representing the age of the
position in seconds.

Range:

```text
0–255 seconds
```

Examples:

```text
0   = position obtained less than approximately 1 second ago
5   = position is approximately 5 seconds old
60  = position is approximately 1 minute old
255 = position is at least 255 seconds old
```

Values greater than the representable range shall saturate at:

```text
255
```

This allows a receiver to determine whether a valid geographic coordinate is
current or stale.

---

## 7. GPS status

Two bits are reserved for position status.

Initial definitions:

| Value | Meaning |
|---|---|
| `00` | No position available |
| `01` | Last known position |
| `10` | Current valid GPS position |
| `11` | Reserved |

A receiver shall not interpret the coordinates as a current GPS fix unless the
status is:

```text
10
```

The exact criteria used by the Garmin application to distinguish a current
position from a last-known position may be refined during implementation.

---

## 8. Reserved bits

Five bits are reserved for future protocol extensions.

Initial value:

```text
00000
```

Transmitters implementing version 1 of this specification shall set all
reserved bits to zero.

Receivers shall ignore the reserved bits.

Possible future uses include:

- protocol flags;
- position source;
- emergency/status indication;
- additional sequencing information.

No meaning is currently assigned to these bits.

---

## 9. Packing

Conceptually, the payload may be constructed as a 64-bit unsigned value:

```text
payload =
      latitude_q
    | (longitude_q << 22)
    | (sequence    << 45)
    | (age         << 49)
    | (gps_status  << 57)
    | (reserved    << 59)
```

The fields must be masked to their allocated width before packing.

Equivalent masks:

```text
latitude_q : 0x3FFFFF
longitude_q: 0x7FFFFF
sequence   : 0x0F
age        : 0xFF
gps_status : 0x03
reserved   : 0x1F
```

This representation defines the logical bit layout only.

The exact mapping of the 64-bit value to the eight ANT payload bytes shall be
defined explicitly after confirming the byte-order requirements of both the
Garmin Connect IQ ANT API and the ESP32 ANT implementation.

No implementation shall rely on native CPU endianness.

---

## 10. Transmission behavior

The initial transmission interval is not yet fixed.

Candidate intervals include:

```text
1 s
5 s
10 s
30 s
60 s
```

The final interval should consider:

- Garmin battery consumption;
- ANT airtime;
- ESP32 power consumption;
- LoRa forwarding interval;
- required tracking responsiveness.

For initial bench testing, a relatively short interval may be used to simplify
debugging.

Field operation may use a significantly longer interval.

---

## 11. Node identification

The ANT position payload does not currently contain a node or user identifier.

Identification should occur outside the position payload whenever possible.

The radio node may associate:

```text
ANT device/channel
        ↓
Local node identity
        ↓
LoRa node identity
```

This avoids repeatedly transmitting identity information in every ANT position
packet.

The final identification mechanism remains to be defined.

---

## 12. LoRa transport

LoRa transport is intentionally not defined by this version of the protocol.

The ANT payload should not be assumed to be identical to the future LoRa
packet.

The radio node may decode the ANT position and construct a different LoRa
message containing additional information such as:

- node ID;
- timestamp;
- relay information;
- message type;
- integrity information;
- network metadata.

This separation allows the ANT and LoRa protocols to evolve independently.

---

## 13. Test vectors

Reference test vectors shall be created before implementation.

Each vector shall contain:

```text
Input latitude
Input longitude
GPS status
Age
Sequence
        ↓
Expected quantized latitude
Expected quantized longitude
        ↓
Expected 64-bit payload
        ↓
Expected 8 ANT bytes
        ↓
Decoded latitude
Decoded longitude
Position error
```

The same vectors shall be used by both:

- the Garmin Connect IQ implementation;
- the ESP32 implementation.

A test passes when both implementations produce identical encoded data and
decode the position within the specified quantization error.

---

## 14. Open questions

The following items remain intentionally undefined:

- ANT payload byte order;
- ANT device number;
- ANT device type;
- ANT transmission type;
- ANT channel period;
- ANT RF channel;
- production position update interval;
- Garmin GPS validity criteria;
- node identification mechanism;
- LoRa protocol;
- bidirectional ANT behavior;
- remote-position message format.

These decisions should be made only after the corresponding hardware behavior
has been experimentally validated.

---

## 15. Protocol development principle

The protocol should remain as simple as possible during the proof-of-concept
phase.

Features shall be added only when they solve a demonstrated requirement.

The first implementation objective is therefore:

```text
GPS position
     ↓
22-bit latitude + 23-bit longitude
     ↓
64-bit position payload
     ↓
ANT
     ↓
ESP32
     ↓
decoded geographic position
```