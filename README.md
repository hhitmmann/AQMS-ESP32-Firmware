# AQMS – ESP32 Firmware

C++ firmware (Arduino framework) for the ESP32 at the heart of my **Air Quality Monitoring System (AQMS)**, my final-year capstone project at NED University.

The firmware reads environmental sensors, shows live data on a TFT display, raises local and remote alerts, and uploads readings to Firebase for the companion Flutter app.

> This repository contains the **ESP32 firmware only**. The complete system (Flutter app, documentation, images, data tables) is in the main project repository:
> 👉 **[Air-Quality-Monitoring-System-AQMS-Project](https://github.com/hhitmmann/Air-Quality-Monitoring-System-AQMS-Project)**

---

## ✨ Features

- **Multi-sensor acquisition**: temperature, humidity, pressure, altitude, CO, CO₂ (estimated), PM1.0 / PM2.5 / PM10 and GPS position
- **Signal smoothing**: 5-sample moving average on the raw MQ-7 and MQ-135 readings
- **Firebase Realtime Database upload** every 60 seconds (`/sensors/current` plus time-ordered history)
- **Offline buffering**: up to 50 readings are queued in a circular buffer while offline and flushed automatically when Wi-Fi and Firebase are back
- **Dual Wi-Fi support**: two saved networks via `WiFiMulti`
- **7-screen TFT display** cycled with a push button (see below)
- **Threshold alerts**: non-blocking buzzer pattern (10 s) plus a Discord webhook message
- **Periodic Discord report** every 5 minutes
- **Local JSON endpoint**: `GET /data` on port 80 returns the current readings
- **GPS with cached fallback**: the last valid position is reused when the fix is lost
- **NTP time sync** so every upload carries a Unix `lastSeen` timestamp

---

## 🔧 Hardware & wiring

| Component | Interface | ESP32 pins |
| --- | --- | --- |
| ST7735 TFT display | SPI | CS 15 · DC 2 · RST 12 · MOSI 23 · SCLK 18 |
| DHT22 (temperature / humidity) | Digital | GPIO 4 |
| BMP280 (pressure / altitude) | I²C, address `0x76` | SDA 16 · SCL 17 |
| MQ-7 (carbon monoxide) | Analog | GPIO 34 |
| MQ-135 (air quality / CO₂ estimate) | Analog | GPIO 35 |
| PMS particulate sensor | UART2, 9600 baud | ESP32 RX = GPIO 26 (from sensor TX) |
| GPS module | UART1, 115200 baud | ESP32 RX = GPIO 27 (from GPS TX) |
| Buzzer | Digital output | GPIO 13 |
| Push button (screen switch) | Input with internal pull-up | GPIO 32 |

---

## 📺 TFT screens

Press the button to cycle through the screens:

1. Temperature and humidity (turns red above the alert threshold)
2. CO (MQ-7)
3. CO₂ (MQ-135)
4. PM1.0 / PM2.5 / PM10 bars
5. Pressure and altitude
6. GPS latitude, longitude and satellite count (marks cached coordinates)
7. System status: alarm state, IP address, database status

---

## 🚨 Alert thresholds

Defined in the sketch as the `KHI_*` constants (tuned for Karachi):

| Parameter | Threshold |
| --- | --- |
| Temperature | ≥ 40 °C |
| Humidity | ≥ 85 % |
| CO | ≥ 9 ppm |
| CO₂ (estimated) | ≥ 800 ppm |
| PM1.0 | ≥ 50 µg/m³ |
| PM2.5 | ≥ 150 µg/m³ |
| PM10 | ≥ 250 µg/m³ |

When a threshold is crossed, the buzzer sounds for 10 seconds and an alert is sent to Discord.

---

## ☁️ Firebase data layout

```
/sensors
  ├── current                         ← latest reading (overwritten each upload)
  └── history
        └── <unixTime>_<millis>       ← one entry per upload, chronologically ordered
```

Each entry contains: `temperature`, `humidity`, `pressure`, `co`, `co2`, `pm1`, `pm25`, `pm10`, `lat`, `lng`, `lastSeen`.

---

## 📁 Project structure

```
AQMS-ESP32-Firmware/
├── README.md
├── .gitignore
└── AQMS_ESP32/
    ├── AQMS_ESP32.ino
    └── config.example.h
```

The sketch folder is named `AQMS_ESP32` to match `AQMS_ESP32.ino`, which the Arduino IDE requires.

---

## ⚙️ Setup

1. Install the **Arduino IDE** and add **ESP32 board support** (Boards Manager), then select your ESP32 board.
2. Install the required libraries (below).
3. Open `AQMS_ESP32/AQMS_ESP32.ino`.
4. In the same folder, copy `config.example.h` to **`config.h`** and fill in your own values:

   | Setting | Purpose |
   | --- | --- |
   | `WIFI_SSID_1` / `WIFI_PASS_1` | Primary Wi-Fi network |
   | `WIFI_SSID_2` / `WIFI_PASS_2` | Secondary (fallback) Wi-Fi network |
   | `discord_webhook` | Discord webhook URL for alerts and reports |
   | `DATABASE_URL` | Firebase Realtime Database URL |
   | `DATABASE_SECRET` | Firebase database secret |

5. Upload to the ESP32 and open the Serial Monitor at **115200 baud**.

> **Important:** `config.h` holds private credentials and is listed in `.gitignore`. Never upload it to GitHub.

### Required libraries

Install these from the Arduino Library Manager (it will offer any dependencies, such as Adafruit Unified Sensor and Adafruit BusIO):

- Adafruit GFX Library
- Adafruit ST7735 and ST7789 Library
- DHT sensor library (Adafruit)
- Adafruit BMP280 Library
- TinyGPSPlus
- PMS Library (fu-hsi)
- ArduinoJson (version 6.x)
- Firebase Arduino Client Library for ESP8266 and ESP32 (Mobizt), which provides `Firebase_ESP_Client.h`

`WiFi`, `WiFiMulti`, `WebServer`, `HTTPClient`, `WiFiClientSecure`, `SPI` and `Wire` come with the ESP32 board package.

---

## 🔌 Local API

When connected to Wi-Fi, the ESP32 serves its current readings at:

```
http://<ESP32-IP>/data
```

The IP address is shown on the status screen. The response is JSON with: `temp`, `hum`, `pressure`, `altitude`, `pm1`, `pm25`, `pm10`, `co`, `co2`, `lat`, `lng`, `alarm_active`, `alarm_type`.

---

## 📝 Notes

- Gas readings come from MQ-series sensors, so CO and CO₂ values are **estimates** derived from a clean-air baseline set in the sketch (`baselineCO_Raw`, `baselineCO2_Raw`). Re-tune these for your own sensors and environment.
- At startup the sketch prints raw GPS data to the Serial Monitor for 15 seconds as a wiring check.

---

## 🔒 Security

This repository intentionally contains no credentials (Wi-Fi passwords, Firebase secrets, webhook URLs). Use `config.example.h` as the template for your own local `config.h`.

---

## 🔮 Possible improvements

- Per-sensor calibration routine run at startup
- Over-the-air (OTA) firmware updates
- Authenticated Firebase access instead of a database secret
- Machine-learning-based air-quality prediction on the collected sensor data

---

## 👨‍💻 Author

**Muhammad Rafay Ali Khan**: [GitHub](https://github.com/hhitmmann) · [LinkedIn](https://www.linkedin.com/in/rafayalikhangg)

`ESP32` · `Arduino` · `C++` · `Embedded Systems` · `IoT` · `Firebase`
