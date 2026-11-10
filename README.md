# ApisMonitor

ApisMonitor is a modular, repairable and energy-autonomous beehive monitoring system built on ESP32-C3 and LoRa. Each hive node measures temperature and humidity inside the brood chamber all year round, without opening the hive and without changing batteries. All data ends up in Home Assistant, so no custom app or backend is required.

> **Status:** design phase. Hardware, firmware and protocol described below are the planned specification; the first prototype (5 assembled boards) is not built yet.

## 🐝 Core Philosophy

Traditional beehive sensors often fail due to propolis buildup, condensation in winter, or batteries that drain within weeks. ApisMonitor addresses this with:

1. **Ultra-Low-Power Design:** Deep sleep between measurements, solar charging and a frost-proof LiFePO4 cell for 365-day operation.
2. **LoRa Transmission:** Tiny sensor payloads sent over LoRa (EU868) with a custom protocol, including relaying through neighbouring nodes if the master is out of direct range.
3. **Propolis- and Condensation-Protected Sensor:** The temperature/humidity sensor has a factory-fitted dust- and water-repellent PTFE membrane and an integrated heater that removes condensation at near 100 % humidity and frost.
4. **Home Assistant Integration:** The master publishes via MQTT discovery. Dashboards, history and push alerts come from Home Assistant and its companion app; remote access (Nabu Casa, Tailscale, ...) is up to the user, no port forwarding is required by the project itself.

## ⚙️ Hardware Architecture

### Master-Slave Topology

- **Slave Nodes (the hives):** ESP32-C3 + SX1262 LoRa. Measure, then send via LoRa. If the master cannot be reached directly, a slave hands its data to a neighbouring slave that can reach it (multi-hop relay, hop-limited).
- **Master Node:** Same board as the slaves, different firmware. Receives all slave data via LoRa, uplinks to Home Assistant over WiFi (MQTT), powered by USB-C. Always listening.
- **Configuration:** Measurement/transmit interval and heater thresholds are set per node via BLE during pairing. Later changes are distributed by the master to the slaves over LoRa.

### Main PCB

- **Form factor:** Rigid 4-layer PCB, assembled by JLCPCB (SMT). Parts are chosen from JLCPCB Basic/Preferred parts wherever possible; Extended parts only where necessary (ESP32 module, SX1262, FHT31, charger IC).
- **Stackup:** JLCPCB standard 4-layer (`JLC04161H-7628`, 1.6 mm). L1 signal/components, L2 continuous GND, L3 3.3 V plane, L4 signal. The 50 Ω RF trace runs on L1 above the unbroken GND plane L2. Trace width to be verified with the JLCPCB impedance calculator.
- **Microcontroller:** ESP32-C3 module (DOIT ESPC3-12-N4, 4 MB flash; final module choice pending Basic/Preferred availability check). RISC-V, native USB for flashing and debugging (GPIO18/19), BLE 5 for pairing and configuration.
- **LoRa:** Silicontra SX1262TR8-GC (868 MHz), BAT WIRELESS BWU.FL-IPEX1 SMD antenna connector (U.FL) to a weatherproof 868 MHz rubber duck antenna outside the housing. *Never transmit without an antenna connected.*
- **User interface:** One button (starts BLE pairing / wakes the node) and one status LED.
- **Battery monitoring:** Voltage divider to an ADC pin (switchable to avoid sleep current).

### Sensor Board (internal)

- **Sensor:** Nyfea FHT31-TR (I2C temperature and humidity, PTFE membrane, programmable heater), on a small separate board.
- **Connection:** Fixed soldered cable (3V3, GND, SDA, SCL) to the main PCB, so the main PCB can stay outside the hive.

### Power Management

- **Battery:** 18650 LiFePO4 (3.2 V nominal), standard cell holder. LiFePO4 is frost-proof, long-lived (3000+ cycles) and safe.
- **Charger:** CONSONANCE CN3058E, CC/CV for LiFePO4 (3.6 V), inputs for a small 5 V solar panel (JST connector) and USB-C. Charging below 0 °C must be prevented (LiFePO4 limitation); how this is done is still to be decided.
- **Regulator:** Buck-boost converter (e.g. TPS63001) for a stable 3.3 V from 2.5–3.65 V cell voltage.

## 🔒 Security

- **Pairing:** Initial onboarding via BLE at the hive, to prevent accidental pairing with neighbouring systems. The shared network key is transferred during pairing.
- **Encryption:** AES-128 network key per installation.
- **Replay Protection:** Authenticated encryption (AEAD, e.g. AES-CCM) with a monotonically increasing message counter.

## 📊 Software & Integration

- **Firmware:** ESP-IDF (C/C++), deep-sleep-centred, SX1262 LoRa stack, BLE configuration, relay logic with hop limit and duplicate detection.
- **Home Assistant:** The master publishes each slave as a device via MQTT discovery (temperature, humidity, battery voltage, signal quality). Alerts and history are handled by Home Assistant.
- **BLE configuration client:** Still open: Web Bluetooth page (Chrome on Android/PC) or a small Flutter app.
- **Duty cycle:** EU868 limits airtime to 1 %, so relaying should stay the exception and payloads small.

## 📂 Repository Structure

    /hardware
      /easyeda_source     # Local EasyEDA project files: main PCB and sensor board
      /production         # Gerber files, BOM, and CPL for manufacturing (JLCPCB)
      /schematics         # Schematics in PDF format
      /mech               # CAD files (.stl, .step) for the enclosure
    /firmware
      /src                # ESP-IDF C/C++ source code (LoRa stack, relay, BLE, mbedTLS)
      /include            # Header files
    /homeassistant        # MQTT discovery examples, dashboards, automations
    /docs
      API.md              # LoRa payload structure documentation
      RESEARCH.md         # Research references (background section below)

---

## 📚 Background Research (earlier acoustic concept)

> The current hardware has no piezo/acoustic sensor. The research below comes from the earlier acoustic concept and is kept as background and as a basis for possible future extensions (temperature-based colony state recognition in sections 2 and 3 stays relevant).

### Temperature & Vibration for Colony Vitality Monitoring
 
Compilation of relevant studies and open datasets as a foundation for the software/ML approach of ApisMonitor. Focus: Findings that can be directly applied to sensor design, sampling parameters, and classification strategy.
 
### 1. Acoustic Frequency Patterns (Reference for Piezo Analysis)
 
| Condition | Observed Pattern | Source |
|---|---|---|
| Night Rest / Winter Cluster | Peaks at 25–150 Hz | Ferrari et al. 2008 |
| Normal Day Activity | Peaks at 100/150–300 Hz | Ferrari et al. 2008 |
| Increased Excitement | Peaks up to 500–600 Hz | Ferrari et al. 2008 |
| Swarm Preparation | Power density rise around 110 Hz, jumping to ~300 Hz shortly before swarming, partial jump 150→500 Hz | Ferrari et al. 2008, *Computers and Electronics in Agriculture* 64 |
| Queenlessness | Additional noise at higher frequencies, day/night rhythm is maintained | SAS4509-2020, "Noninvasive Beehive Monitoring through Acoustic Data" |
| Varroa Infestation | Acoustic signature detectable, but less strongly validated | Referenced in Terenzi et al., various review articles |
 
### 2. Combined Temperature + Vibration
 
- **Swarm-Temperature Correlation:** Internal temperature rises from approx. 33 °C to 35 °C before the swarm leaves and drops to approx. 32 °C at the actual time of departure — parallel to the known acoustic frequency rise.
  *Source:* cited in Kulyukin & Mukherjee, *Applied Sciences* 8(9):1573, "Toward Audio Beehive Monitoring", referring to Ferrari et al. 2008.
  → **Implication for Software:** Temperature and piezo signals should be treated as a fused feature combination, not analyzed separately — a simultaneous occurrence of temperature rise + frequency shift is a much more robust swarm signal than either channel alone.
- **Vibration Sensors on the Outer Hive Wall:** Ramsey et al. analyzed swarm events using accelerometers mounted on the outside of the hive wall (not between the frames) — methodological confirmation that structure-borne vibration measurement (solid-borne sound through the hive wall) works for swarm detection.
  *Source:* Referenced in Kulyukin & Mukherjee, *Applied Sciences* 8(9):1573.
- **Pure Vibration Classification Over a Full Season:** A system with vibration sensors on the brood frames (Warsaw-type hives) collected 9 months of data and used it to classify five lifecycle phases of the colony using CNN and Extra-Trees models. Key finding: crucial frequencies are below 1 kHz; a sampling rate of 2 kHz is sufficient without significant loss of accuracy (>95% weighted accuracy).
  *Source:* *Applied Sciences* 16(1):188, "Honey Bee Lifecycle Activity Prediction Using Non-Invasive Vibration Monitoring".
  → **Implication:** Confirms that a low sampling rate/computational load on the ESP is sufficient.

### 3. Temperature-Only Approaches (Fallback Model Without Acoustics)
 
- **Growth Phase Recognition from Temperature Time Series:** Multivariate temperature feature extraction (various time scales) + PCA for dimensionality reduction, six ML models compared; a BP (Backpropagation) network achieved the best performance (MAE 1.45%).
  *Source:* *Sensors* 25(13):3916, "Honeybee Colony Growth Period Recognition Based on Multivariate Temperature Feature Extraction and Machine Learning".
  → **Implication:** Useful as a fallback classification in case the piezo channel fails.
- **Temperature Drop Forecasting (LSTM):** Predicting temperature drops 24h in advance using LSTM, trained on internal temperature, humidity, fanning activity, noise, weight, and external temperature (RMSE 0.5%).
  *Source:* *Computers and Electronics in Agriculture*, "Forecasting sudden drops of temperature in pre-overwintering honeybee colonies" (Data basis: Arnia monitoring system).
  → **Implication:** A reduced variant (only internal/external temp + piezo activity instead of all 6 inputs) is conceivable as an early-warning model for winter.

### 4. Important Methodological Limitation
 
Several studies show good classification accuracy on the same hives already seen during training, but **poor generalization to unknown, new colonies** (explicitly discussed in Uthoff et al. 2023 and the BeeTogether study, among others).
 
→ **Consequence for App Architecture:** Classification should be performed as anomaly detection against a baseline learned individually per hive (first 1–2 weeks post-installation as reference period), rather than relying on a globally pre-trained, universal model.
 
### 5. Open Datasets (for Pre-training / Benchmarking Before Own Season Data)
 
| Dataset | Scope | Modalities | Source |
|---|---|---|---|
| **UrBAN** | >3000 h audio, 10 hives | Audio, temperature, humidity, Varroa/winter loss labels | Nature *Scientific Data* 12, 536 (2025), "Urban Beehive Acoustics and PheNotyping Dataset" |
| **MSPB** | 53 hives, Québec | Audio, temperature, humidity | Nature *Scientific Data*, "MSPB: a longitudinal multi-sensor dataset with phenotypic trait measurements" |
| **NU-Hive** | 2 hives, 96 h audio | Audio, temperature, humidity, queenrightness labeled | Referenced in several of the above studies |
| **BUZZ** | 6 hives | Audio (bee hum/cricket/noise) | Referenced in TinyML Survey (arXiv 2509.08822) |
| **BeeTogether** | Consolidation of multiple open audio datasets | Standardized format, data augmentation | Kaggle, *Sensors* 24(18):6067 |
 
### 6. Reference List (Short Form)
 
- Ferrari, S. et al. (2008). *Monitoring of swarming sounds in bee hives for early detection of the swarming period*. Computers and Electronics in Agriculture, 64, 72–77.
- Uthoff, C., Homsi, M. N., von Bergen, M. (2023). *Acoustic and vibration monitoring of honeybee colonies for beekeeping-relevant aspects of presence of queen bee and swarming*. Computers and Electronics in Agriculture, 205, 107589.
- Kulyukin, V., Mukherjee, S. (2018). *Toward Audio Beehive Monitoring: Deep Learning vs. Standard Machine Learning in Classifying Beehive Audio Samples*. Applied Sciences, 8(9), 1573.
- *Honey Bee Lifecycle Activity Prediction Using Non-Invasive Vibration Monitoring*. Applied Sciences, 16(1), 188.
- *Honeybee Colony Growth Period Recognition Based on Multivariate Temperature Feature Extraction and Machine Learning*. Sensors, 25(13), 3916.
- *Forecasting sudden drops of temperature in pre-overwintering honeybee colonies*. Computers and Electronics in Agriculture (Arnia Dataset).
- *UrBAN: Urban Beehive Acoustics and PheNotyping Dataset*. Scientific Data, 12, 536 (2025).
- *MSPB: a longitudinal multi-sensor dataset with phenotypic trait measurements from honey bees*. Scientific Data (2024).
- *Bee Together: Joining Bee Audio Datasets for Hive Extrapolation in AI-Based Monitoring*. Sensors, 24(18), 6067 (2024).
- SAS4509-2020. *Noninvasive Beehive Monitoring through Acoustic Data*. SAS Institute.
