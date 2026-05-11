# Twin-Source Weather Logger: Edge-to-Cloud IoT + Data Science

**End-to-end sensor collection, cloud monitoring, and offline-first logging for data analysis.**

## 1) Project Summary
This project builds a complete pipeline from **edge sensor readings** (DHT22 temperature & humidity) to **cloud monitoring** (Arduino IoT Cloud dashboard) while simultaneously logging **timestamped CSV rows** to an SD card for **Python-based analytics**.

Two data sources are captured at a fixed interval (10 minutes):
- **Local sensor**: DHT22 → current indoor temperature (°F) and humidity (%RH)
- **Web weather**: wttr.in JSON endpoint → current outdoor temperature (°F) and humidity for a ZIP code

## 2) System Architecture (Edge → Cloud → Offline Analytics)
**Edge device**: Wemos D1 R32 (ESP32-class MCU) + DHT22 + data logging shield (RTC + SD)

**Data flow**
1. The board connects to Wi‑Fi.
2. It syncs time using NTP (network time).
3. Every 10 minutes it:
   - Reads temperature/humidity from DHT22
   - Fetches ZIP weather via HTTP GET (returns JSON)
   - Updates Arduino IoT Cloud properties
   - Logs records to SD card as CSV

**Why both Arduino IoT Cloud and SD logging?**
- Cloud dashboard gives *real-time visibility* and remote monitoring.
- SD card provides *offline-first raw data* for reproducible analysis even if connectivity drops.

## 3) Data Transmission & Protocols (Short Description)
This project intentionally touches several real-world protocols across the full stack:

### Network & cloud
- **Wi‑Fi (802.11)** provides the local wireless link from the device to the Internet.
- **Arduino IoT Cloud** uses open standards for data exchange such as **MQTT** and **SenML**, and its IoT library implements an **X.509 certificate-based chain of trust** for device authentication. (Arduino’s docs describe these design choices.)
- The sketch updates cloud “properties” via the `ArduinoCloud.update()` loop.

### Web weather API
- The device fetches weather from `wttr.in/<zipcode>?format=j1` using **HTTP GET**, then parses the returned **JSON** payload to extract `temp_F` and `humidity`.

### Time
- The sketch uses **NTP** (via `configTime`) to obtain accurate timestamps.
- An onboard **RTC (DS1307)** can also provide timekeeping; it communicates via **I²C**.

### Local peripherals
- **SD card logging** uses the Arduino `SD` library; SD cards are accessed over the **SPI** bus.
- **DHT22** uses a timing-based **single-wire digital protocol** (custom, not I²C/SPI/UART).

## 4) Why this Hardware Was a Good Choice
**Wemos D1 R32 (ESP32-class)** plus an SD/RTC shield is ideal for exploring an end-to-end monitoring system because it combines:
- **Built-in Wi‑Fi** for cloud connectivity and API calls.
- **Enough compute** for periodic sensing, JSON parsing, and cloud updates.
- **Removable storage (SD)** for robust, offline logging and later analysis.
- **Reliable time sources** (NTP + RTC option) for correctly timestamped data.
- **Beginner-friendly ecosystem**: Arduino IDE + libraries (DHT, SD, ArduinoIoTCloud) make integration fast, so you can focus on system concepts.

## 5) CSV Logging Format
Your sketch writes a **long/tidy** log shape:

`date_time,date,hour,minutes,seconds,sensor_name,measurement,sensor_value,Units`

Example measurements:
- `DHT22,Temperature,<value>,F`
- `DHT22,Humidity,<value>,%`
- `web_weather_wttr.in,web_temp75024,<value>,F`
- `web_weather_wttr.in,web_humidity75024,<value>,%`

This is excellent for analytics because it is:
- Append-only (streaming-friendly)
- Easy to group/pivot by `measurement`

## 6) Proposed Analyses (Python + Pandas + Seaborn + Streamlit)
Below are analysis ideas that teach core data analysis concepts using *your* real data.

### A. Data quality & cleaning fundamentals
- Parse timestamps; check for missing intervals.
- Identify and remove outliers (e.g., impossible humidity > 100%).
- Validate sensor sampling constraints (DHT22 has recommended minimum intervals).

### B. Indoor vs outdoor comparison (feature engineering)
- Compute **delta**: `indoor_temp - outdoor_temp` and `indoor_humidity - outdoor_humidity`.
- Track delta distributions over days/weeks.
- Detect **bias** and **drift** (is the sensor consistently reading high/low?).

### C. Time-series exploration
- Rolling mean / rolling std to smooth noise.
- Daily seasonality (hour-of-day patterns).
- Week-over-week comparisons.

### D. Correlation & relationships
- Correlation between temperature and humidity (indoor and outdoor separately).
- Scatter plots with regression lines.
- Lag correlation: does outdoor change lead indoor change after some lag?

### E. Anomaly detection (simple + interpretable)
- Z-score or IQR-based anomaly flags on deltas.
- Change-point heuristics: sudden jumps between consecutive readings.

### F. Derived comfort metrics (applied analytics)
- Heat Index (uses temp + RH)
- Dew Point estimate
- “Comfort zones” classification (e.g., RH 30–60% typical indoor comfort)

### G. Dashboarding & storytelling
- A Streamlit dashboard with:
  - Time-series plots
  - Indoor vs outdoor deltas
  - Anomaly table
  - Downloadable filtered CSV

## 7) Repo Layout (suggested)
```
.
├─ arduino/
│  └─ wemos_d1_r32_weather_logger.ino
├─ data/
│  └─ sensor_log.csv
├─ notebooks/
│  └─ exploration.ipynb   # optional
├─ streamlit_app.py
├─ analysis_starter.py
├─ requirements.txt
└─ README.md
```

## 8) Running the Analytics
### Install
```bash
pip install -r requirements.txt
```

### Run quick analysis
```bash
python analysis_starter.py --csv data/sensor_log.csv
```

### Run dashboard
```bash
streamlit run streamlit_app.py
```

---

## Security note (important)
Do **not** publish Wi‑Fi credentials or Arduino IoT Cloud device secrets in public repos. Use placeholders and keep secrets in a local `.env` or separate config file.
