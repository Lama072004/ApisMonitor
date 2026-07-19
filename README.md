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
```

# Forschungsgrundlage: Temperatur & Vibration für Volksvitalitäts-Monitoring
 
Zusammenstellung relevanter Studien und offener Datensätze als Grundlage für den Software-/ML-Ansatz von ApisMonitor. Fokus: Erkenntnisse, die sich direkt auf Sensordesign, Sampling-Parameter und Klassifikationsstrategie übertragen lassen.
 
## 1. Akustische Frequenzmuster (Referenz für Piezo-Auswertung)
 
| Zustand | Beobachtetes Muster | Quelle |
|---|---|---|
| Nachtruhe / Wintercluster | Peaks bei 25–150 Hz | Ferrari et al. 2008 |
| Normale Tagaktivität | Peaks bei 100/150–300 Hz | Ferrari et al. 2008 |
| Erhöhte Erregung | Peaks bis 500–600 Hz | Ferrari et al. 2008 |
| Schwarmvorbereitung | Anstieg der Leistungsdichte um ~110 Hz, kurz vor Auszug Anstieg auf ~300 Hz, teils Sprung 150→500 Hz | Ferrari et al. 2008, "Monitoring of swarming sounds in bee hives for early detection of the swarming period", *Computers and Electronics in Agriculture* 64 |
| Weisellosigkeit | Zusätzliches Rauschen bei höheren Frequenzen, Tag/Nacht-Rhythmus bleibt erhalten | SAS4509-2020, "Noninvasive Beehive Monitoring through Acoustic Data" |
| Varroa-Befall | Akustische Signatur nachweisbar, aber schwächer validiert | referenziert in Terenzi et al., div. Übersichtsarbeiten |
 
## 2. Temperatur + Vibration kombiniert
 
- **Schwarm-Temperaturkorrelation**: Innentemperatur steigt vor dem Schwarmauszug von ca. 33 °C auf 35 °C und fällt zum eigentlichen Auszugzeitpunkt auf ca. 32 °C — parallel zum bekannten akustischen Frequenzanstieg. Quelle: zitiert in Kulyukin & Mukherjee, *Applied Sciences* 8(9):1573, "Toward Audio Beehive Monitoring", mit Bezug auf Ferrari et al. 2008.
  → **Implikation für die Software**: Temperatur- und Piezo-Signal sollten als fusionierte Feature-Kombination behandelt werden, nicht getrennt ausgewertet — ein gleichzeitiges Auftreten von Temperaturanstieg + Frequenzverschiebung ist ein deutlich robusteres Schwarm-Signal als jeder Kanal einzeln.
- **Vibrationssensoren an der Zargenaußenwand**: Ramsey et al. analysierten Schwarmereignisse über Beschleunigungssensoren, die außen an der Zargenwand montiert waren (nicht zwischen den Waben) — methodische Bestätigung, dass strukturgebundene Vibrationsmessung (Körperschall durch die Beutenwand) für Schwarmerkennung funktioniert. Referenziert in Kulyukin & Mukherjee, *Applied Sciences* 8(9):1573.
- **Reine Vibrationsklassifikation über volle Saison**: Ein System mit Vibrationssensoren auf den Brutwaben (Warschau-Typ-Beuten) erfasste 9 Monate Daten und klassifizierte damit fünf Lebenszyklusphasen des Volkes über CNN- und Extra-Trees-Modelle. Wichtiges Ergebnis: entscheidende Frequenzen liegen unter 1 kHz, Sampling-Rate von 2 kHz reicht ohne nennenswerten Genauigkeitsverlust (>95 % gewichtete Genauigkeit). Quelle: *Applied Sciences* 16(1):188, "Honey Bee Lifecycle Activity Prediction Using Non-Invasive Vibration Monitoring".
  → **Implikation**: bestätigt niedrige Sampling-Rate/Rechenlast auf dem ESP als ausreichend.
## 3. Temperatur-only Ansätze (Fallback-Modell ohne Akustik)
 
- **Wachstumsphasen-Erkennung aus Temperatur-Zeitreihen**: Multivariate Temperatur-Feature-Extraktion (verschiedene Zeitskalen) + PCA zur Dimensionsreduktion, sechs ML-Modelle verglichen; ein BP-Netz (Backpropagation) erzielte die beste Performance (MAE 1,45 %). Quelle: *Sensors* 25(13):3916, "Honeybee Colony Growth Period Recognition Based on Multivariate Temperature Feature Extraction and Machine Learning".
  → Nützlich als Fallback-Klassifikation, falls der Piezo-Kanal ausfällt.
- **Temperaturabfall-Vorhersage (LSTM)**: Vorhersage von Temperaturabfällen 24 h im Voraus mit LSTM, trainiert auf Innentemperatur, Feuchte, Fächelaktivität, Lärm, Gewicht, Außentemperatur (RMSE 0,5 %). Quelle: *Computers and Electronics in Agriculture*, "Forecasting sudden drops of temperature in pre-overwintering honeybee colonies" (Datenbasis: Arnia-Monitoringsystem).
  → Reduzierte Variante (nur Innen-/Außentemp + Piezo-Aktivität statt aller 6 Eingaben) als Winter-Frühwarnmodell denkbar.
## 4. Wichtige methodische Einschränkung
 
Mehrere Studien zeigen gute Klassifikationsgenauigkeit auf denselben, bereits im Training gesehenen Stöcken, aber **schlechte Generalisierung auf unbekannte, neue Völker** (u.a. explizit diskutiert in Uthoff et al. 2023 und in der BeeTogether-Studie). 
→ **Konsequenz für die App-Architektur**: Klassifikation sollte als Anomalieerkennung gegen eine pro Stock individuell gelernte Baseline erfolgen (erste 1–2 Wochen nach Installation als Referenzzeitraum), nicht als global vortrainiertes, universelles Modell.
 
## 5. Offene Datensätze (für Vortraining / Benchmarking vor eigenen Saisondaten)
 
| Dataset | Umfang | Modalitäten | Link/Quelle |
|---|---|---|---|
| **UrBAN** | >3000 h Audio, 10 Stöcke | Audio, Temperatur, Feuchte, Varroa-/Winterausfall-Labels | Nature *Scientific Data* 12, 536 (2025), "Urban Beehive Acoustics and PheNotyping Dataset" |
| **MSPB** | 53 Stöcke, Québec | Audio, Temperatur, Feuchte | Nature *Scientific Data*, "MSPB: a longitudinal multi-sensor dataset with phenotypic trait measurements" |
| **NU-Hive** | 2 Stöcke, 96 h Audio | Audio, Temperatur, Feuchte, Weiselrichtigkeit gelabelt | referenziert in mehreren o.g. Studien |
| **BUZZ** | 6 Stöcke | Audio (Bienensummen/Grille/Rauschen) | referenziert in TinyML-Survey (arXiv 2509.08822) |
| **BeeTogether** | Zusammenführung mehrerer offener Audio-Datensätze | Standardisiertes Format, Data Augmentation | Kaggle, *Sensors* 24(18):6067 |
 
## 6. Referenzliste (Kurzform)
 
- Ferrari, S. et al. (2008). *Monitoring of swarming sounds in bee hives for early detection of the swarming period*. Computers and Electronics in Agriculture, 64, 72–77.
- Uthoff, C., Homsi, M. N., von Bergen, M. (2023). *Acoustic and vibration monitoring of honeybee colonies for beekeeping-relevant aspects of presence of queen bee and swarming*. Computers and Electronics in Agriculture, 205, 107589.
- Kulyukin, V., Mukherjee, S. (2018). *Toward Audio Beehive Monitoring: Deep Learning vs. Standard Machine Learning in Classifying Beehive Audio Samples*. Applied Sciences, 8(9), 1573.
- (2026). *Honey Bee Lifecycle Activity Prediction Using Non-Invasive Vibration Monitoring*. Applied Sciences, 16(1), 188.
- (2025). *Honeybee Colony Growth Period Recognition Based on Multivariate Temperature Feature Extraction and Machine Learning*. Sensors, 25(13), 3916.
- *Forecasting sudden drops of temperature in pre-overwintering honeybee colonies*. Computers and Electronics in Agriculture (Arnia-Datensatz).
- *UrBAN: Urban Beehive Acoustics and PheNotyping Dataset*. Scientific Data, 12, 536 (2025).
- *MSPB: a longitudinal multi-sensor dataset with phenotypic trait measurements from honey bees*. Scientific Data (2024).
- *Bee Together: Joining Bee Audio Datasets for Hive Extrapolation in AI-Based Monitoring*. Sensors, 24(18), 6067 (2024).
- SAS4509-2020. *Noninvasive Beehive Monitoring through Acoustic Data*. SAS Institute.
---
*Hinweis: Diese Zusammenfassung dient als Ausgangspunkt für den ApisMonitor-Software-Ansatz. Für die konkrete Implementierung sollten die Originalarbeiten (insb. Methodik zu Feature-Extraktion und Modellwahl) im Volltext konsultiert werden.*
