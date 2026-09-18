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
