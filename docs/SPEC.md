# Meshtastic Solar Deer Camera — Design Spec

## Overview

A solar-powered trail camera that captures wildlife photos and transmits them over a Meshtastic LoRa mesh network. Designed for remote locations with no cell service or WiFi. Camera triggers on motion, stores locally, and forwards thumbnails/images over the mesh to a receiving node.

---

## System Architecture

```
[PIR Sensor] → [ESP32-CAM / Trail Cam] → [Meshtastic Node] → [LoRa Mesh] → [Receiver Node] → [Phone/Laptop]
        ↑                                           ↑
   [Solar Panel] → [Charge Controller] → [LiFePO4 Battery] → [All components]
```

### Two design approaches:

**A) Integrated Build (DIY)**
- ESP32-CAM module + Meshtastic firmware (or custom)
- PIR motion sensor triggers photo
- Sends JPEG over Meshtastic mesh
- Lower cost, more control, more work

**B) Commercial Trail Cam + Meshtastic Relay**
- Commercial trail camera (Reolink, Browning, etc.) with SD card
- Separate Meshtastic node co-located
- ESP32 reads SD card or camera's output, forwards via Meshtastic
- Less custom firmware, more hardware cost

**Recommendation:** Start with Approach A (integrated ESP32-CAM) for proof of concept.

---

## Hardware Components

### Core
| Component | Part | Est. Cost | Notes |
|-----------|------|-----------|-------|
| MCU + Camera | ESP32-CAM module | $8-12 | OV2640 sensor, 2MP, has microSD slot |
| LoRa Radio | LilyGO T-Echo or Heltec LoRa 32 V3 | $25-35 | 915MHz (US), has display + battery jack |
| Or combined | LilyGO T-Camera Plus | $30-40 | ESP32 + camera + LoRa in one board |
| PIR Sensor | AM312 or HC-SR501 | $2-5 | Low-power AM312 preferred (28µA standby) |
| Solar Panel | 5W-10W 6V panel | $10-20 | Size for winter/cloudy days |
| Charge Controller | TP4056 + solar input OR CN3791 | $2-5 | CN3791 is MPPT, better efficiency |
| Battery | 18650 LiFePO4 (2-3 cells) | $8-15 | LiFePO4 safer than LiPo, better cold weather |
| Battery Holder | 18650 holder (2-3 cell) | $2 | |
| Weatherproof Box | IP67 junction box | $8-15 | Need to drill holes for camera lens + PIR |
| Antenna | 915MHz whip or dipole | $5-10 | External antenna for better range |

### Optional / Nice-to-Have
| Component | Part | Cost | Notes |
|-----------|------|------|-------|
| MicroSD | 32GB SD card | $5 | Local storage fallback |
| Voltage Monitor | INA226 breakout | $3 | Battery/solar monitoring |
| IR LEDs | 850nm or 940nm IR LED array | $3-5 | Night vision (940nm = no visible glow) |
| IR Cut Filter | ESP32-CAM with IR-cut | +$3 | Better daytime colors if using IR LEDs |
| Level Shifter | 3.3V↔5V level shifter | $2 | If mixing voltage levels |

### Total Estimated Cost: $60-120 (depending on approach)

---

## Power Budget

### Power Consumption
| Component | Active | Standby | Duty Cycle |
|-----------|--------|---------|------------|
| ESP32-CAM (taking photo) | ~200mA | ~10mA (deep sleep ~0.15mA) | 1-5% |
| LoRa Radio (transmitting) | ~120mA | ~5mA (idle) / ~0.1mA (sleep) | <1% |
| PIR Sensor (AM312) | ~0.028mA | 0.028mA | 100% |
| Charge Controller | ~0.5mA | 0.5mA | 100% |

### Solar Sizing (worst case — December, cloudy US)
- Average daily consumption: ~50-100mAh (mostly sleep, few photos per day)
- 5W panel in December: ~200-400mAh/day even with clouds
- 2x 18650 LiFePO4 (3000mAh each = 6000mAh total): ~60 days reserve with no sun

**Verdict:** 5W panel + 2x 18650 LiFePO4 is overkill. Even 3W would work. 10W gives margin for IR night vision.

---

## Software / Firmware

### Meshtastic Integration

Meshtastic firmware supports custom plugins via the `meshtastic/firmware` repo. The camera node needs a custom plugin that:

1. **Listens for PIR trigger** (GPIO interrupt)
2. **Wakes from deep sleep** on PIR interrupt
3. **Captures photo** via ESP32-CAM
4. **Saves full image** to SD card (local backup)
5. **Generates thumbnail** (160x120 or 320x240 JPEG, ~2-10KB)
6. **Sends over Meshtastic** using `sendImage` or custom packet type
7. **Returns to deep sleep**

### Image Transmission Strategy

LoRa bandwidth is extremely limited (~250 bytes/sec at default settings). Options:

| Strategy | Size | Time to Send | Notes |
|----------|------|-------------|-------|
| Full JPEG (VGA 640x480) | ~30-80KB | 2-5 min | Too slow for frequent triggers |
| Thumbnail (160x120) | ~2-5KB | 10-20 sec | Good for "something was here" alerts |
| Thumbnail (320x240) | ~8-15KB | 30-60 sec | Better detail, still reasonable |
| Text alert only | ~50 bytes | <1 sec | "Motion detected at Camera 1" — no image |
| Chunked full image | 30-80KB | 2-5 min | Send in chunks over several minutes |

**Recommendation:** Send a 320x240 thumbnail + "motion detected" alert. Store full image on SD for physical retrieval.

### Firmware Options

**Option 1: Custom ESP32-CAM firmware + Meshtastic serial client**
- ESP32-CAM runs custom camera code
- Connects to a Heltec/LilyGO Meshtastic node via serial (UART)
- Sends image data as serial packets, Meshtastic node forwards over mesh
- Pro: Clean separation, Meshtastic firmware stays stock
- Con: Two boards, more power, more wiring

**Option 2: Custom Meshtastic plugin on ESP32-CAM**
- Fork Meshtastic firmware, add camera plugin
- Single board does everything
- Pro: Single device, lower power, cleaner
- Con: Maintaining a fork, Meshtastic firmware is large for ESP32-CAM

**Option 3: ESP32-CAM + Meshtastic API over WiFi (local)**
- ESP32-CAM hosts a tiny web server
- Separate Meshtastic node polls for new images and forwards
- Pro: No custom firmware on Meshtastic side
- Con: WiFi range limited (co-location only), more complexity

**Recommendation:** Option 1 for prototype (separate boards), Option 2 for final integrated version.

---

## Receiver Side

On the receiving end (the node connected to a phone/laptop):

1. **Meshtastic app** (phone) receives the image packets
2. Custom plugin or Python script reassembles chunks into a JPEG
3. Displays in app or saves to phone gallery
4. Optional: forwards to Discord/webhook/email for remote monitoring

### Python Receiver Script (example flow)
```
meshtastic --serial    # connect to receiver node
# Listen for IMAGE packets (portnum 70 or custom)
# Reassemble chunks
# Save to /photos/YYYY-MM-DD_HH-MM-SS.jpg
# Optional: POST to Discord webhook
```

---

## Enclosure Design

### Requirements
- Weatherproof (IP67 minimum)
- Camera lens window (clear acrylic or glass)
- PIR sensor window (thin plastic, PIR sees through thin plastic)
- Solar panel mounted on top or external via cable
- Antenna external (through waterproof cable gland)
- Easy battery access for maintenance

### Layout
```
┌─────────────────────────────┐
│  [Solar Panel on top]       │
│  ┌───────────────────────┐  │
│  │ Battery │ ESP32-CAM   │  │
│  │  x2     │ + LoRa      │  │
│  │         │ PIR →──┐    │  │
│  │ Charge  │ SD    │    │  │
│  │ Ctrl    │       │    │  │
│  └───────────────────────┘  │
│  [Lens] [PIR]  [Antenna]    │
└─────────────────────────────┘
```

### Mounting
- Tree strap or bungee cord mount
- ~3-4 ft height for deer-level shots
- Tilt slightly downward (10-15°) to avoid sky exposure
- Face North (so sun doesn't blind camera)

---

## Meshtastic Network Configuration

### Camera Node Settings
```yaml
# Channel config
channel:
  name: "deer-cam"
  psk: <shared key>
  
# LoRa config
lora:
  region: US915
  modem_preset: LongFast  # or LongModerate for more range
  
# Power
power:
  is_power_saving: true
  sleep_timeout: 300       # 5 min after activity
  disable_leds: true       # save power
  
# Telemetry
telemetry:
  interval: 3600           # battery report every hour
```

### Mesh Range Expectations
| Environment | Range | Notes |
|-------------|-------|-------|
| Open field | 5-15 km | Line of sight |
| Forest | 1-3 km | Trees attenuate LoRa |
| Hills/valleys | 0.5-2 km | Depends on terrain |
| With repeater node | doubles+ | Place on hilltop |

If the camera is too far from the receiver, add a solar-powered repeater node on a hilltop between them.

---

## Bill of Materials (Prototype)

| # | Item | Source | Est. Cost |
|---|------|--------|-----------|
| 1 | ESP32-CAM module | Amazon/AliExpress | $10 |
| 2 | Heltec LoRa 32 V3 (915MHz) | Amazon/AliExpress | $30 |
| 3 | AM312 PIR sensor | Amazon | $3 |
| 4 | 5W 6V solar panel | Amazon | $12 |
| 5 | CN3791 MPPT charge module | AliExpress | $4 |
| 6 | 2x 18650 LiFePO4 batteries | Amazon | $12 |
| 7 | 18650 2-cell holder | Amazon | $2 |
| 8 | IP67 junction box (120x80x60mm) | Amazon | $10 |
| 9 | 915MHz external antenna + cable | Amazon | $8 |
| 10 | MicroSD 32GB | Amazon | $5 |
| 11 | Jumper wires + solder | — | $3 |
| 12 | Waterproof cable glands | Amazon | $3 |
| | **Total** | | **~$102** |

---

## Development Phases

### Phase 1: Proof of Concept (Week 1-2)
- ESP32-CAM + PIR sensor on breadboard
- Write firmware: PIR trigger → capture photo → save to SD
- Test camera quality, trigger speed, power consumption
- No LoRa yet — just verify camera + sensor works

### Phase 2: Add Meshtastic (Week 2-3)
- Connect Heltec LoRa node via UART
- Send thumbnail over Meshtastic mesh
- Test range between camera node and receiver
- Verify image reassembly on receiver side

### Phase 3: Solar Power (Week 3-4)
- Add solar panel + charge controller + batteries
- Test deep sleep current draw
- Verify multi-day operation on solar alone
- Optimize duty cycle

### Phase 4: Weatherproof + Deploy (Week 4-5)
- Final enclosure build
- Cable glands for antenna and sensor
- Mount in field, test for real
- Monitor battery levels via Meshtastic telemetry

### Phase 5: Polish (ongoing)
- Custom Meshtastic plugin (integrated firmware)
- Discord/webhook forwarding
- Multiple camera nodes
- Time-lapse mode (interval capture, not just motion)
- Thermal sensor option (MLX90640) for heat detection

---

## Known Limitations & Trade-offs

1. **Image quality:** ESP32-CAM is 2MP, decent in daylight, noisy at night. Not trail-cam quality. For better images, use a real trail cam + relay (Approach B).

2. **Transmission speed:** LoRa is SLOW. A 10KB thumbnail takes ~30-60 seconds to send. Don't expect real-time video or high-res images over mesh.

3. **Trigger latency:** ESP32-CAM wakeup + capture takes ~1-2 seconds. Fast-moving animals may be gone before photo is taken. PIR sensor detects them, but camera is slow to wake.

4. **Night vision:** IR LEDs will draw significant power. May need larger solar panel or separate IR illumination. ESP32-CAM without IR-cut filter sees IR, but daytime colors are washed out.

5. **Cold weather:** LiFePO4 handles cold better than LiPo, but below -10°C capacity drops. Solar panels also produce less in winter. Size for worst case.

6. **Meshtastic firmware size:** ESP32-CAM has 4MB flash. Meshtastic firmware + camera code may be tight. May need to strip features or use a board with more flash.

---

## Alternatives & Upgrades

| Idea | Benefit | Cost |
|------|---------|------|
| Use RAK WisBlock instead of ESP32-CAM | Better Meshtastic support, lower power | $40-50 |
| Add OV5640 camera (5MP) | Better image quality | +$10 |
| Use ESP32-S3 with PSRAM | More memory for image processing | $12 |
| Starlink Mini + real trail cam | Real-time high-res, but $$$ | $600+ |
| Cellular trail cam (Moultrie/SPYPOINT) | Easy, but monthly fee + cell coverage needed | $150 + $10/mo |
| Add RTC (DS3231) | Accurate timestamps on photos | $2 |
| Add temperature sensor (BME280) | Environmental data alongside photos | $3 |

---

## References

- [Meshtastic Documentation](https://meshtastic.org/docs/)
- [Meshtastic Firmware GitHub](https://github.com/meshtastic/firmware)
- [ESP32-CAM Arduino Examples](https://github.com/espressif/arduino-esp32/tree/master/libraries/ESP32/examples/Camera)
- [ESP32 Deep Sleep Guide](https://randomnerdtutorials.com/esp32-deep-sleep-arduino-ide-wake-up-sources/)
- [LoRa Range Calculator](https://www.loratools.com/)
- [LiFePO4 Solar Charging with CN3791](https://www.cn3791.com/)