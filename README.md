# ApisMonitor
 
ApisMonitor is a modular, repairable beehive monitoring system built with ESP32-C5 and LoRa. It tracks hive acoustics via a piezo element and measures cluster temperature to help detect swarming or queen loss. To ensure long-term durability, the design deliberately avoids propolis-prone gas or humidity sensors.
 
## 🐝 Core Philosophy
 
Traditional beehive scales and sensors often fail due to propolis buildup or drain batteries within weeks because of heavy WiFi usage. ApisMonitor solves this through:
 
1. **Edge Computing:** The ESP32-C5 performs Fast Fourier Transform (FFT) directly on the node, extracting acoustic peaks.
2. **LoRa Transmission:** Instead of raw audio, only a tiny 30–60 byte feature payload is sent via LoRa (EU868) every 30–60 minutes.
3. **Propolis-Proof Sensors:** No gas, humidity, or VOC sensors. All internal sensors are fully potted or rely on structural vibration.

## ⚙️ Hardware Architecture
 
### Master-Slave Topology
 
- **Master Node:** ESP32-C5 + SX1262 LoRa Module + WiFi Uplink (MQTT/HTTPS to Docker Backend).
- **Slave Nodes (The Hives):** ESP32-C5 + SX1262 LoRa Module. Communicates exclusively via LoRa.

### Sensor Flex-PCB (Internal)
 
The sensors are mounted on a minimal flex-PCB (max 3×3 cm) that simply hangs over the edge of the brood box. No drilling required.
 
- **Acoustics:** Murata 7BB-12-9 piezo element (12 mm). Potted between two rigid pressure distribution plates to withstand the static weight of the supers (~60 kg) without bending — the failure mode of concern is uneven bending stress, not compressive strength.
- **Temperature:** DS18B20 (1-Wire), fully potted. Measures the brood nest/winter cluster temperature.
- **Light:** Photodiode to detect when the hive is opened.

### Power Management (Ultra-Low-Power)
 
- **Battery:** 18650 LiFePO4 (3.2 V, 1500 mAh) — extremely cold-resistant and safe.
- **Regulator:** High-efficiency buck-boost converter (e.g., TPS630xx) providing stable 3.3 V, as LiFePO4 drops from 3.65 V to 2.5 V.
- **Charging:** CC/CV charging (via ICs like CN3058 or LTC4162-L) connected to a small 5 V solar panel. No trickle charging, to ensure battery longevity.

## 🔒 Security
 
- **Pairing:** Initial slave onboarding is done via Bluetooth (BLE) at the hive to prevent accidental pairing with neighboring systems.
- **Encryption:** Individual AES-128 session keys per node.
- **Replay Protection:** AEAD encryption (AES-CCM/ChaCha20-Poly1305) with a monotonically increasing counter.

## 📊 Software & Backend
 
The backend runs on a custom Docker stack (MQTT Broker, InfluxDB).
 
- **No Fixed Thresholds:** Swarm or queenlessness detection is not based on static Hz thresholds. The backend uses machine learning to establish a baseline for each specific hive during the first 1–2 weeks.
- **Known Acoustic Patterns:**
  - *Night/Cluster:* Peaks at 25–150 Hz
  - *Normal Day Activity:* Peaks at 100–300 Hz
  - *Swarm Prep:* Power density rise around 110 Hz, jumping to ~300 Hz shortly before swarming
  - *Queenlessness:* High-frequency noise, subtle volume shifts

## 📂 Repository Structure
 
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
      API.md              # LoRa payload structure documentation
      RESEARCH.md         # Research references (this section, extracted as standalone file)
 
---
 
## 📚 Research Foundation: Temperature & Vibration for Colony Vitality Monitoring
 
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
