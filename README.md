# ApisMonitor
ApisMonitor is a modular, repairable beehive monitoring system built with ESP32-C5 and LoRa. It tracks hive acoustics via a piezo microphone and measures cluster temperature to help detect swarming or queen loss. To ensure long-term durability, the design deliberately avoids propolis-prone gas or humidity sensors.

## 🐝 Core Philosophy
Traditional beehive scales and sensors often fail due to propolis buildup or drain batteries within weeks because of heavy WiFi usage. ApisMonitor solves this through:
1. **Edge Computing:** The ESP32-C5 performs Fast Fourier Transform (FFT) directly on the node, extracting acoustic peaks.
2. **LoRa Transmission:** Instead of raw audio, only a tiny 30-60 byte feature payload is sent via LoRa (EU868) every 30-60 minutes.
3. **Propolis-Proof Sensorik:** No gas, humidity, or VOC sensors. All internal sensors are fully potted or rely on structural vibration.

## ⚙️ Hardware Architecture

### Master-Slave Topology
*   **Master Node:** ESP32-C5 + SX1262 LoRa Module + WiFi Uplink (MQTT/HTTPS to Docker Backend).
*   **Slave Nodes (The Hives):** ESP32-C5 + SX1262 LoRa Module. Communicates exclusively via LoRa.

### Sensor Flex-PCB (Internal)
The sensors are mounted on a minimal flex-PCB (max 3x3 cm) that simply hangs over the edge of the brood box. No drilling required.
*   **Acoustics:** Murata 7BB-12-9 Piezo element (12mm). Potted between two rigid pressure distribution plates to withstand the weight of the supers (~60 kg) without bending.
*   **Temperature:** DS18B20 (1-Wire), fully potted. Measures the brood nest/winter cluster temperature.
*   **Light:** Photodiode to detect when the hive is opened.

### Power Management (Ultra-Low-Power)
*   **Battery:** 18650 LiFePO4 (3.2V, 1500mAh) – extremely cold-resistant and safe.
*   **Regulator:** High-efficiency Buck-Boost Converter (e.g., TPS630xx) providing stable 3.3V, as LiFePO4 drops from 3.65V to 2.5V.
*   **Charging:** CC/CV charging (via ICs like CN3058 or LTC4162-L) connected to a small 5V solar panel. No trickle charging to ensure battery longevity.

## 🔒 Security
*   **Pairing:** Initial slave onboarding is done via Bluetooth (BLE) at the hive to prevent accidental pairing with neighboring systems.
*   **Encryption:** Individual AES-128 session keys per node.
*   **Replay Protection:** AEAD encryption (AES-CCM/ChaCha20-Poly1305) with a monotonically increasing counter. 

## 📊 Software & Backend
The backend runs on a custom Docker stack (MQTT Broker, InfluxDB).
*   **No Fixed Thresholds:** Swarm or queenlessness detection is not based on static Hz thresholds. The backend uses Machine Learning to establish a baseline for each specific hive during the first 1-2 weeks.
*   **Known Acoustic Patterns:**
    *   *Night/Cluster:* Peaks at 25–150 Hz
    *   *Normal Day Activity:* Peaks at 100–300 Hz
    *   *Swarm Prep:* Power density rise around 110 Hz, jumping to ~300 Hz shortly before swarming.
    *   *Queenlessness:* High-frequency noise, subtle volume shifts.

## 📂 Repository Structure

```text
/hardware
  /easyeda_source     # Exported EasyEDA project files (.json) for Main-PCB & Flex-PCB
  /production         # Gerber files, BOM, and CPL for manufacturing (e.g., JLCPCB)
  /schematics         # Schematics in PDF format
  /mech               # CAD files (.stl, .step) for enclosures and piezo pressure plates
/firmware
  /src                # ESP-IDF C/C++ source code (FFT logic, LoRa stack, mbedTLS)
  /include            # Header files
/backend
  /docker             # Docker-Compose setup (Mosquitto, InfluxDB)
  /app                # Python ML scripts and classification logic
/docs
  API.md              # LoRa Payload structure documentation
