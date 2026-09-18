# fenix-ant-lora

Experimental position-sharing system using Garmin fēnix watches, ANT and LoRa.

## Objective

The goal of this project is to extend the capabilities of compatible Garmin
fēnix watches without replacing their existing navigation functions.

A Garmin watch provides its GPS position to a nearby ESP32-based radio node
using ANT. The node can then transmit that position over LoRa to other nodes.

```text
Garmin fēnix
     │
     │ ANT
     ▼
 ESP32-S3
     │
     │ LoRa
     ▼
Other nodes
```

The long-term goal is bidirectional communication, allowing the watch to
display information received from other LoRa nodes.

## Initial hardware

- Garmin fēnix 3
- Heltec WiFi LoRa 32 V3
  - ESP32-S3
  - SX1262 LoRa transceiver

## Project status

Early proof of concept.

The first milestone is to establish bidirectional ANT communication between
a Garmin fēnix 3 and an ESP32-S3.

## Planned development

- [ ] Validate ANT reception on ESP32-S3
- [ ] Transmit ANT data from fēnix 3
- [ ] Establish bidirectional ANT communication
- [ ] Transfer GPS coordinates over ANT
- [ ] Forward position data over LoRa
- [ ] Receive positions from remote nodes
- [ ] Display remote position information on the watch
- [ ] Develop a wearable radio-node enclosure

## Architecture

The system is designed so that the Garmin watch remains fully usable as a
standalone navigation device. The external radio node adds position-sharing
capabilities without replacing the watch's native navigation functions.

```text
                     ANT
┌──────────────┐   2.4 GHz   ┌────────────────────┐
│ Garmin       │◄───────────►│ ESP32-S3           │
│ fēnix        │             │                    │
│              │             │ ANT ↔ LoRa bridge  │
│ GPS          │             │ SX1262             │
│ Navigation   │             └─────────┬──────────┘
└──────────────┘                       │
                                      │ LoRa
                                      ▼
                              Remote radio nodes
```

The radio node is optional. If it is disconnected or powered off, the Garmin
watch continues to operate normally.

## Development strategy

Development will proceed incrementally:

1. Validate ANT operation on the ESP32-S3.
2. Establish one-way communication from the fēnix to the ESP32.
3. Establish bidirectional ANT communication.
4. Transfer GPS position data.
5. Add LoRa communication.
6. Exchange positions between multiple nodes.
7. Display remote position information on the watch.
8. Develop and optimize wearable hardware.

Each communication layer should be independently testable before integration
with the next layer.

## License

This project is licensed under the MIT License.