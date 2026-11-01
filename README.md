# Meshtastic Solar Deer Camera

Solar-powered trail camera that captures wildlife photos and transmits them over a [Meshtastic](https://meshtastic.org) LoRa mesh network. Designed for remote locations with no cell service or WiFi.

## How It Works

1. PIR motion sensor detects movement
2. ESP32-CAM wakes from deep sleep and captures a photo
3. Full image saves to SD card (local backup)
4. Thumbnail (320x240) transmits over LoRa mesh to a receiver node
5. Receiver forwards to phone, laptop, or Discord webhook
6. Solar panel keeps batteries charged indefinitely

## Documentation

- [Full Design Spec](docs/SPEC.md) — hardware, firmware, power budget, BOM, phased dev plan

## Bill of Materials (~$102)

See [SPEC.md](docs/SPEC.md) for full BOM and sourcing links.

## Status

📋 Planning phase — spec complete, firmware development not yet started.

## License

MIT
