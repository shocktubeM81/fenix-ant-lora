# System Architecture

## Purpose

The system extends the navigation capabilities of a compatible Garmin fēnix
watch by adding long-range position sharing.

The Garmin watch remains a fully functional standalone navigation device.
The external radio node is optional and does not replace the native Garmin
navigation functions.

## High-level architecture

```text
                    Local link
                  ANT / 2.4 GHz

┌─────────────────┐              ┌─────────────────────┐
│ Garmin fēnix    │─────────────►│ Radio node          │
│                 │              │                     │
│ GPS             │              │ ESP32-S3            │
│ Navigation      │              │ SX1262 LoRa         │
│ User interface  │              │ Battery             │
└─────────────────┘              └──────────┬──────────┘
                                           │
                                           │ LoRa
                                           │
                                 ┌─────────▼───────────┐
                                 │ Remote radio nodes │
                                 └─────────────────────┘
```

## Garmin watch

### Responsibilities

- Obtain the user's GPS position
- Retain native Garmin navigation capabilities
- Transmit position information over ANT
- Eventually display information received from remote nodes

### Initial target

Garmin fēnix 3.

The architecture should avoid unnecessary dependencies on this specific
model so that other ANT-capable Garmin devices may eventually be supported.

## Radio node

### Initial hardware

Heltec WiFi LoRa 32 V3:

- ESP32-S3
- SX1262 LoRa transceiver

### Responsibilities

- Receive ANT messages from the watch
- Decode position information
- Transmit position information over LoRa
- Receive LoRa messages from other nodes
- Eventually forward remote information to the watch

The initial prototype does not require Bluetooth or Wi-Fi.

## Communication layers

### Watch ↔ radio node

ANT is used for short-range communication between the Garmin watch and the
radio node.

Initial development will focus on:

```text
fēnix ──ANT──► ESP32-S3
```

Bidirectional ANT communication will be investigated separately.

### Radio node ↔ radio network

LoRa will provide the long-range radio link.

The LoRa protocol has not yet been selected.

Possible approaches include:

- Custom LoRa protocol
- Existing mesh protocol
- Other compatible implementations

This decision will be made after the ANT proof of concept.

## Design principles

1. The Garmin watch must remain useful without the radio node.
2. Failure of the radio node must not affect native navigation.
3. ANT and LoRa should remain separate communication layers.
4. Each layer should be independently testable.
5. The first prototypes should prioritize validation over optimization.
6. Hardware miniaturization will occur only after the communication
   architecture has been validated.

## Development phases

### Phase 1 — ANT proof of concept

```text
fēnix 3 ──ANT──► ESP32-S3 ──USB──► Serial console
```

Goal: receive a known ANT payload transmitted by the fēnix.

### Phase 2 — Position transfer

```text
fēnix GPS ──► ANT ──► ESP32-S3
```

Goal: transfer and reconstruct actual GPS coordinates.

### Phase 3 — LoRa transport

```text
fēnix ──ANT──► Node A ──LoRa──► Node B
```

Goal: transmit the position between two radio nodes.

### Phase 4 — Remote information

Investigate communication in the opposite direction:

```text
Remote node ──LoRa──► ESP32-S3 ──ANT──► fēnix
```

Goal: allow the watch to display information about remote nodes.

### Phase 5 — Wearable hardware

Develop a compact battery-powered radio node suitable for mounting on
equipment such as a helmet.

## Open questions

- Can the ESP32-S3 reliably receive ANT transmissions from the fēnix 3?
- Can the ESP32-S3 reliably transmit ANT messages accepted by the fēnix 3?
- What ANT channel configuration should be used?
- What position encoding should be used?
- What position update interval provides the best trade-off between tracking
  quality and power consumption?
- Should the LoRa layer use a custom protocol or an existing mesh protocol?
- What range can be achieved with a wearable antenna configuration?
- What battery capacity is required for practical field use?