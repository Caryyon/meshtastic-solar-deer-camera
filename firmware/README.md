# Firmware

ESP32-CAM camera firmware + Meshtastic integration.

## Structure (planned)

- `firmware/camera/` — ESP32-CAM camera capture + PIR trigger + deep sleep
- `firmware/meshtastic-plugin/` — Custom Meshtastic plugin for image transmission
- `firmature/receiver/` — Python receiver script for image reassembly

See [docs/SPEC.md](../docs/SPEC.md) for firmware architecture details.
