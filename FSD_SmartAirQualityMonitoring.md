# Functional Specification Document (FSD)
## Smart Air Quality Monitoring System
**Versi:** 1.0.0  
**Tanggal:** 2025-01-01  
**Status:** Draft  
**Platform:** ESP32 DevKit + MQTT over WiFi

---

## Daftar Isi
1. [Overview Sistem](#1-overview-sistem)
2. [Block Diagram](#2-block-diagram)
3. [Komponen & Spesifikasi](#3-komponen--spesifikasi)
4. [Pinout & Wiring](#4-pinout--wiring)
5. [Arsitektur Software](#5-arsitektur-software)
6. [File Structure](#6-file-structure)
7. [Protokol MQTT](#7-protokol-mqtt)
8. [Logika Kontrol Aktuator](#8-logika-kontrol-aktuator)
9. [Non-Blocking Code Policy](#9-non-blocking-code-policy)
10. [Do & Don't](#10-do--dont)
11. [Error Handling](#11-error-handling)
12. [Parameter Konfigurasi](#12-parameter-konfigurasi)

---

## 1. Overview Sistem

Smart Air Quality Monitoring adalah sistem embedded untuk memantau kualitas udara lingkungan secara real-time menggunakan ESP32 sebagai mikrokontroler utama. Sistem membaca data dari dua sensor (debu dan gas), mengolahnya, lalu mempublikasikan data melalui protokol MQTT ke aplikasi client. Dua buah fan (intake & exhaust) dikendalikan sebagai aktuator berdasarkan kondisi kualitas udara.

```
Tujuan Sistem:
- Membaca kadar debu (µg/m³) dari sensor GP2Y1010AU0F
- Membaca kadar gas/VOC/CO₂ dari sensor MQ135 (ppm)
- Mempublikasikan data sensor ke MQTT broker secara periodik
- Mengontrol fan intake & exhaust berdasarkan threshold kualitas udara
- Menerima perintah kontrol manual via MQTT dari aplikasi client
```

---

## 2. Block Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        POWER SUPPLY                                 │
│                    5V DC (USB / Adaptor)                            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
          ┌────────────────────▼────────────────────┐
          │              ESP32 DevKit                │
          │          (MCU / Otak Utama)              │
          │                                          │
          │  ┌──────────┐     ┌────────────────┐    │
          │  │  WiFi    │     │  MQTT Client   │    │
          │  │  Stack   │────▶│  (PubSubClient)│    │
          │  └──────────┘     └────────────────┘    │
          │        │                  │              │
          │        ▼                  ▼              │
          │  ┌──────────┐     ┌──────────────┐      │
          │  │  AP/STA  │     │ MQTT Broker  │      │
          │  │  Mode    │     │  (External)  │      │
          │  └──────────┘     └──────────────┘      │
          └──────┬───────────────────────────────────┘
                 │
    ┌────────────┼────────────┐─────────────────────┐
    │            │            │                     │
    ▼            ▼            ▼                     ▼
┌───────┐  ┌─────────┐  ┌─────────┐         ┌──────────┐
│MQ135  │  │GP2Y1010 │  │  FAN    │         │  FAN     │
│Gas    │  │ Dust    │  │  INTAKE │         │  EXHAUST │
│Sensor │  │ Sensor  │  │(MOSFET/ │         │ (MOSFET/ │
│(ADC)  │  │(ADC+GPIO│  │  Relay) │         │  Relay)  │
└───────┘  └─────────┘  └─────────┘         └──────────┘

Analog    Analog+Pulse   Digital PWM          Digital PWM
Input     Input+Trigger  Output               Output
```

### Data Flow Diagram

```
[GP2Y1010] ──(analog)──▶ [ADC CH1] ──▶ [Voltage→Dust Calc] ──▶ ┐
                                                                  ├──▶ [Data Aggregator]
[MQ135]    ──(analog)──▶ [ADC CH2] ──▶ [Voltage→PPM Calc]  ──▶ ┘
                                                                        │
                                                                        ▼
                                                               [Air Quality Index]
                                                                        │
                            ┌───────────────────────────────────────────┤
                            │                                           │
                            ▼                                           ▼
                   [MQTT Publisher]                           [Fan Controller]
                   topic: aq/sensor/data                      Intake + Exhaust
                            │
                            ▼
                   [MQTT Broker]
                            │
                            ▼
                   [Aplikasi Client]
                   (Subscribe & Control)
                            │
                            ▼ (command)
                   [MQTT Subscriber]
                   topic: aq/control/fan
                            │
                            ▼
                   [Fan Controller] ◀────── Override Manual
```

---

## 3. Komponen & Spesifikasi

### 3.1 Mikrokontroler

| Parameter | Detail |
|-----------|--------|
| **Modul** | ESP32 DevKit v1 / v4 |
| **WiFi** | 802.11 b/g/n (2.4 GHz) |
| **ADC** | 12-bit SAR ADC, 18 channel |
| **GPIO** | 34 pin (input/output) |
| **Operating Voltage** | 3.3V logic, 5V supply via USB |

> ⚠️ **PENTING:** ADC2 pada ESP32 **tidak bisa digunakan** saat WiFi aktif. Gunakan **ADC1** saja (GPIO32–GPIO39).

### 3.2 Sensor Debu — SHARP GP2Y1010AU0F

| Parameter | Nilai |
|-----------|-------|
| **Supply Voltage** | 5V DC |
| **LED Drive Current** | 11 mA (via 150Ω resistor) |
| **Output Type** | Analog voltage |
| **Output Range** | 0 – 3.5V (linier terhadap debu) |
| **Sensitivity** | 0.5V / (0.1 mg/m³) |
| **Pulse Duration (LED ON)** | 0.32 ms (320 µs) |
| **Sampling Delay** | 0.28 ms (280 µs) setelah LED ON |
| **Measurement Range** | 0 – 0.5 mg/m³ |
| **Pin 1 (V-LED)** | Anoda LED + 150Ω ke 5V |
| **Pin 2 (LED-GND)** | Katoda LED ke GND |
| **Pin 3 (LED)** | Kontrol LED (GPIO ESP32) |
| **Pin 4 (S-GND)** | Signal GND |
| **Pin 5 (Vo)** | Output analog → ADC ESP32 |
| **Pin 6 (Vcc)** | 5V supply |

> ⚠️ **PENTING:** Output GP2Y1010 adalah **0–3.5V**. Gunakan **voltage divider** (10kΩ + 15kΩ) untuk menurunkan ke 0–2.1V agar aman untuk ADC ESP32 yang maksimal **3.3V**. Atau gunakan resistor 3.3V logic level shifter.

**Formula Konversi:**
```
dustDensity (mg/m³) = (Vout - 0.0) / 5.0
// Formula yang lebih akurat (SHARP Application Note):
// if Vout >= 0.6V:
//   dustDensity = (Vout - 0.0) / 5.0  [mg/m³ → µg/m³ × 1000]
// Referensi: 0.9V ≈ 0.18 mg/m³ = 180 µg/m³
```

### 3.3 Sensor Gas — MQ135

| Parameter | Nilai |
|-----------|-------|
| **Supply Voltage** | 5V DC |
| **Heater Voltage** | 5V |
| **Heater Resistance** | 33Ω ± 5% |
| **Heater Consumption** | < 800 mW |
| **Output Type** | Analog (0–5V) + Digital threshold |
| **Target Gases** | NH₃, NOx, Benzene, CO₂, Alkohol, Asap |
| **Range** | 10–300 ppm (NH₃), 10–1000 ppm (CO₂) |
| **Preheat Time** | ≥ 48 jam (burn-in), 20 menit operasional |

> ⚠️ **PENTING:** Output MQ135 adalah **0–5V**. **Wajib** menggunakan voltage divider (10kΩ + 10kΩ) untuk membagi ke 0–2.5V sebelum masuk ADC ESP32.

**Formula Konversi (Simplified):**
```
// Rs = sensor resistance
// Ro = resistance pada udara bersih (dikalibrasikan)
// ratio = Rs / Ro
// ppm = a * pow(ratio, b)   // a, b dari datasheet curve fitting
```

### 3.4 Aktuator — Dual Fan

| Parameter | Nilai |
|-----------|-------|
| **Jumlah** | 2 unit (Intake + Exhaust) |
| **Tegangan Operasi** | 5V atau 12V DC (sesuai spesifikasi fan) |
| **Driver** | MOSFET (IRLZ44N/IRF520) atau Relay Module |
| **Kontrol** | PWM atau ON/OFF Digital |
| **Kontrol Pin** | GPIO ESP32 (PWM capable) |
| **Proteksi** | Dioda flyback (1N4007) antiparallel pada fan |

---

## 4. Pinout & Wiring

### 4.1 Tabel Pinout ESP32

```
┌──────────────────────────────────────────────────────────────────┐
│                        ESP32 DevKit v1                           │
├────────────┬───────────┬───────────────────────────────────────── ┤
│ GPIO       │ Pin Label │ Fungsi                                   │
├────────────┼───────────┼───────────────────────────────────────── ┤
│ GPIO 34    │ D34 (ADC1_6) │ [INPUT] GP2Y1010 – Vout (Analog)    │
│ GPIO 32    │ D32 (ADC1_4) │ [INPUT] MQ135 – Aout (Analog)       │
│ GPIO 25    │ D25       │ [OUTPUT] GP2Y1010 – LED Control (Digital)│
│ GPIO 26    │ D26 (PWM) │ [OUTPUT] Fan Intake – PWM / ON-OFF      │
│ GPIO 27    │ D27 (PWM) │ [OUTPUT] Fan Exhaust – PWM / ON-OFF     │
│ GPIO 2     │ D2        │ [OUTPUT] Onboard LED – Status Indicator  │
│ 3.3V       │ 3V3       │ [POWER] Logic power (tidak untuk sensor) │
│ 5V (VIN)   │ VIN       │ [POWER] 5V untuk sensor & fan driver     │
│ GND        │ GND       │ [GROUND] Common ground                   │
└────────────┴───────────┴───────────────────────────────────────── ┘
```

### 4.2 Wiring Diagram — GP2Y1010AU0F

```
                GP2Y1010AU0F
               ┌──────────────┐
    5V ────────┤ Pin 1 (VLED) │──── 150Ω ──── 5V
    GND ───────┤ Pin 2 (GND)  │
GPIO25 ────────┤ Pin 3 (LED)  │   (Active LOW — LOW = LED ON)
    GND ───────┤ Pin 4 (S-GND)│
GPIO34 ◀───────┤ Pin 5 (Vo)   │──── Voltage Divider ──┐
    5V ────────┤ Pin 6 (VCC)  │                        │
               └──────────────┘               10kΩ ──┬─ ke GPIO34
                                                      │
                                              15kΩ ──┴─ ke GND
                                    (Vout_max 3.5V × 15/(10+15) = 2.1V ✓)
```

> **Kapasitor Filter:** Tambahkan 220µF elektrolit + 0.1µF ceramic paralel antara VCC dan GND sensor untuk menstabilkan power saat LED aktif (arus spike 11mA).

### 4.3 Wiring Diagram — MQ135

```
                  MQ135 Module
               ┌──────────────┐
    5V ────────┤ VCC          │
    GND ───────┤ GND          │
[Tidak dipakai]┤ DOUT         │   (Digital threshold — tidak digunakan)
GPIO32 ◀───────┤ AOUT         │──── Voltage Divider ──┐
               └──────────────┘                        │
                                              10kΩ ──┬─ ke GPIO32
                                                      │
                                              10kΩ ──┴─ ke GND
                                    (Vout_max 5.0V × 10/(10+10) = 2.5V ✓)
```

### 4.4 Wiring Diagram — Fan Driver (MOSFET)

```
                                    Fan Intake (5V/12V)
    GPIO26 ──── Gate               ┌──────────────┐
                │                  │      +        │──── 5V/12V
               [IRLZ44N]          ┤ Fan Motor    ├
                │                  │      -        │──┐
               Drain ─────────────┘               │  │
                │                                  │  │
               Source ──── GND        1N4007 ──────┘  │
                                    (flyback, katoda ke + supply)

    GPIO27 ──── Gate    (sama, untuk Fan Exhaust)
```

> Gunakan **resistor 100Ω** antara GPIO dan Gate MOSFET untuk membatasi arus dan mencegah osilasi. Pull-down 10kΩ dari Gate ke GND untuk memastikan fan mati saat ESP32 boot/reset.

---

## 5. Arsitektur Software

### 5.1 State Machine Utama

```
                    ┌──────────┐
               ┌───▶│   INIT   │◀───────────────────┐
               │    └──────────┘                     │
               │         │ setup() selesai           │
               │         ▼                           │ WiFi/MQTT lost
               │    ┌──────────┐                     │
               │    │ CONNECT  │─── retry > 5 ───────┤
               │    └──────────┘                     │
               │         │ WiFi + MQTT connected     │
               │         ▼                           │
               │    ┌──────────┐                     │
               │    │   IDLE   │─── disconnected ────┘
               │    └──────────┘
               │         │ timer interval
               │         ▼
               │    ┌──────────┐
               │    │  SAMPLE  │ (baca ADC GP2Y1010 + MQ135)
               │    └──────────┘
               │         │
               │         ▼
               │    ┌──────────┐
               │    │ PROCESS  │ (kalkulasi µg/m³, ppm, AQI)
               │    └──────────┘
               │         │
               │         ▼
               │    ┌──────────┐
               │    │ PUBLISH  │ (kirim JSON ke MQTT)
               │    └──────────┘
               │         │
               │         ▼
               │    ┌──────────┐
               └────│ACTUATE   │ (update fan speed berdasarkan AQI)
                    └──────────┘
```

### 5.2 Timer Architecture (Non-Blocking)

```
Ticker/millis-based timers:
┌─────────────────────────────────────────────────────────────┐
│  SENSOR_INTERVAL    = 2000ms    → baca & publish sensor     │
│  RECONNECT_INTERVAL = 5000ms    → coba reconnect WiFi/MQTT  │
│  HEARTBEAT_INTERVAL = 30000ms   → publish status/uptime     │
│  DUST_LED_PULSE     = 280µs     → sampling window GP2Y1010  │
│  DUST_LED_ON_TIME   = 320µs     → LED aktif GP2Y1010        │
└─────────────────────────────────────────────────────────────┘
```

> ⚠️ `DUST_LED_PULSE` dan `DUST_LED_ON_TIME` yang sangat singkat (< 1ms) membutuhkan `micros()` bukan `millis()`, dan **tidak boleh** menggunakan `delayMicroseconds()` dalam konteks loop utama. Gunakan ISR atau flag-based polling.

---

## 6. File Structure

```
smart-air-quality/
│
├── smart-air-quality.ino          # Entry point Arduino (setup + loop)
│
├── config/
│   ├── config.h                   # Semua konstanta & konfigurasi
│   └── secrets.h                  # WiFi SSID, Password, MQTT creds (JANGAN di-commit)
│
├── src/
│   ├── sensors/
│   │   ├── DustSensor.h           # Class/interface GP2Y1010
│   │   ├── DustSensor.cpp         # Implementasi: init, trigger, read, convert
│   │   ├── GasSensor.h            # Class/interface MQ135
│   │   └── GasSensor.cpp          # Implementasi: init, read, calibrate, toPPM
│   │
│   ├── actuators/
│   │   ├── FanController.h        # Class Fan (intake + exhaust)
│   │   └── FanController.cpp      # Implementasi: init, setSpeed, setMode (auto/manual)
│   │
│   ├── connectivity/
│   │   ├── WiFiManager.h          # WiFi connection handler
│   │   ├── WiFiManager.cpp        # Auto reconnect, status callback
│   │   ├── MQTTClient.h           # MQTT wrapper (PubSubClient)
│   │   └── MQTTClient.cpp         # connect, publish, subscribe, callback
│   │
│   ├── processing/
│   │   ├── AirQualityIndex.h      # AQI calculation & kategorisasi
│   │   ├── AirQualityIndex.cpp    # Formula AQI dari dust + gas
│   │   ├── DataPayload.h          # JSON builder untuk MQTT publish
│   │   └── DataPayload.cpp        # serialize sensor data ke JSON string
│   │
│   └── utils/
│       ├── Logger.h               # Serial debug logger (bisa di-disable)
│       ├── Logger.cpp
│       ├── TimerManager.h         # Non-blocking timer helper (millis-based)
│       └── TimerManager.cpp       # checkInterval(), resetTimer()
│
├── test/
│   ├── test_dust_sensor.cpp       # Unit test DustSensor (dengan mock ADC)
│   ├── test_gas_sensor.cpp        # Unit test GasSensor
│   ├── test_aqi.cpp               # Unit test AQI calculation
│   └── test_payload.cpp           # Unit test JSON serialization
│
├── docs/
│   ├── FSD_SmartAirQualityMonitoring.md   # Dokumen ini
│   ├── wiring_diagram.png         # Gambar skematik wiring
│   └── mqtt_topics.md             # Dokumentasi MQTT topic & payload
│
├── .gitignore                     # Exclude secrets.h, build artifacts
├── platformio.ini                 # PlatformIO config (board, lib_deps)
└── README.md                      # Quick start guide
```

### 6.1 config.h — Contoh Isi

```cpp
#pragma once

// ── Pin Definitions ──────────────────────────────────────────
#define PIN_DUST_AOUT       34      // GP2Y1010 Vout → ADC1_CH6
#define PIN_DUST_LED        25      // GP2Y1010 LED control
#define PIN_GAS_AOUT        32      // MQ135 Aout → ADC1_CH4
#define PIN_FAN_INTAKE      26      // Fan Intake PWM
#define PIN_FAN_EXHAUST     27      // Fan Exhaust PWM
#define PIN_STATUS_LED      2       // Onboard LED

// ── Sensor Timing (microseconds) ─────────────────────────────
#define DUST_LED_ON_US      320     // LED aktif 320µs
#define DUST_SAMPLE_DELAY_US 280    // Baca ADC 280µs setelah LED ON

// ── Sampling Interval (milliseconds) ─────────────────────────
#define SENSOR_READ_INTERVAL_MS     2000
#define RECONNECT_INTERVAL_MS       5000
#define HEARTBEAT_INTERVAL_MS       30000

// ── ADC & Conversion ─────────────────────────────────────────
#define ADC_RESOLUTION      4095.0f // 12-bit
#define ADC_VREF            3.3f    // 3.3V reference
#define DUST_VDIVIDER_RATIO 1.4286f // (10k+15k)/15k — kembalikan ke tegangan asli
#define GAS_VDIVIDER_RATIO  2.0f    // (10k+10k)/10k

// ── Air Quality Thresholds (µg/m³ untuk PM2.5 equivalent) ────
#define AQI_GOOD_MAX        35.4f   // Baik
#define AQI_MODERATE_MAX    75.4f   // Sedang
#define AQI_UNHEALTHY_MAX   150.4f  // Tidak Sehat

// ── Fan PWM ──────────────────────────────────────────────────
#define FAN_PWM_CHANNEL_INTAKE  0
#define FAN_PWM_CHANNEL_EXHAUST 1
#define FAN_PWM_FREQ        25000   // 25 kHz (inaudible)
#define FAN_PWM_RESOLUTION  8       // 8-bit (0–255)
#define FAN_SPEED_OFF       0
#define FAN_SPEED_LOW       80      // ~31%
#define FAN_SPEED_MED       160     // ~63%
#define FAN_SPEED_HIGH      255     // 100%

// ── MQTT ─────────────────────────────────────────────────────
#define MQTT_TOPIC_SENSOR_DATA  "aq/sensor/data"
#define MQTT_TOPIC_STATUS       "aq/device/status"
#define MQTT_TOPIC_FAN_CONTROL  "aq/control/fan"
#define MQTT_TOPIC_FAN_STATUS   "aq/fan/status"
#define MQTT_CLIENT_ID          "esp32-airmon-01"
#define MQTT_QOS                1
```

### 6.2 secrets.h — Template (JANGAN commit ke repo)

```cpp
#pragma once
// ⚠️ Tambahkan file ini ke .gitignore

#define WIFI_SSID       "YourSSID"
#define WIFI_PASSWORD   "YourPassword"
#define MQTT_BROKER_IP  "192.168.1.100"
#define MQTT_PORT       1883
#define MQTT_USERNAME   "mqttuser"
#define MQTT_PASSWORD   "mqttpass"
```

---

## 7. Protokol MQTT

### 7.1 Topic Structure

```
aq/
├── sensor/
│   └── data          → [PUBLISH] Data sensor periodik (JSON)
├── device/
│   └── status        → [PUBLISH] Status device / heartbeat (JSON)
├── control/
│   └── fan           → [SUBSCRIBE] Perintah kontrol fan dari client (JSON)
└── fan/
    └── status        → [PUBLISH] Status fan aktual setelah perubahan (JSON)
```

### 7.2 Payload Format

**`aq/sensor/data` — Sensor Data (Publish, setiap 2 detik)**
```json
{
  "device_id": "esp32-airmon-01",
  "timestamp": 1704067200,
  "dust": {
    "raw_voltage": 1.24,
    "density_ugm3": 45.2,
    "unit": "µg/m³"
  },
  "gas": {
    "raw_voltage": 1.85,
    "ppm": 412.5,
    "unit": "ppm"
  },
  "aqi": {
    "value": 62,
    "category": "MODERATE",
    "color": "#FFFF00"
  },
  "fan": {
    "intake_speed": 160,
    "exhaust_speed": 160,
    "mode": "AUTO"
  }
}
```

**`aq/device/status` — Heartbeat (Publish, setiap 30 detik)**
```json
{
  "device_id": "esp32-airmon-01",
  "uptime_s": 3600,
  "wifi_rssi": -65,
  "free_heap": 245760,
  "status": "ONLINE"
}
```

**`aq/control/fan` — Fan Control Command (Subscribe)**
```json
{
  "mode": "MANUAL",
  "intake_speed": 200,
  "exhaust_speed": 200
}
```
```json
{
  "mode": "AUTO"
}
```
```json
{
  "mode": "OFF"
}
```

**`aq/fan/status` — Fan Status Response (Publish)**
```json
{
  "device_id": "esp32-airmon-01",
  "mode": "MANUAL",
  "intake_speed": 200,
  "exhaust_speed": 200,
  "ack": true
}
```

### 7.3 QoS & Retain Policy

| Topic | QoS | Retain |
|-------|-----|--------|
| `aq/sensor/data` | 0 (Fire & Forget) | false |
| `aq/device/status` | 1 (At Least Once) | true |
| `aq/control/fan` | 1 | false |
| `aq/fan/status` | 1 | true |

---

## 8. Logika Kontrol Aktuator

### 8.1 Mode Operasi Fan

```
Mode AUTO   → Fan dikontrol otomatis berdasarkan AQI
Mode MANUAL → Fan dikontrol via perintah MQTT dari client
Mode OFF    → Semua fan mati
```

### 8.2 Tabel Auto Control

| Kondisi | Dust (µg/m³) | Gas (ppm) | Fan Intake | Fan Exhaust |
|---------|-------------|-----------|------------|-------------|
| **GOOD** | 0 – 35.4 | 0 – 300 | OFF (0) | OFF (0) |
| **MODERATE** | 35.4 – 75.4 | 300 – 500 | LOW (80) | LOW (80) |
| **UNHEALTHY** | 75.4 – 150.4 | 500 – 800 | MED (160) | MED (160) |
| **HAZARDOUS** | > 150.4 | > 800 | HIGH (255) | HIGH (255) |

> Kondisi diambil dari **nilai terburuk** antara dust dan gas (worst-case logic).

---

## 9. Non-Blocking Code Policy

### 9.1 Prinsip Utama

> **DILARANG KERAS menggunakan `delay()` atau `delayMicroseconds()` di dalam `loop()`.**
> Semua timing harus menggunakan pola **millis/micros + flag**.

### 9.2 Pattern Wajib — Timer Millis

```cpp
// ✅ BENAR — Non-blocking timer
unsigned long lastSensorRead = 0;

void loop() {
    unsigned long now = millis();

    if (now - lastSensorRead >= SENSOR_READ_INTERVAL_MS) {
        lastSensorRead = now;
        readAndPublishSensors();
    }

    mqttClient.loop();   // MQTT keep-alive — WAJIB dipanggil tiap loop
    wifiManager.loop();  // WiFi reconnect check
}
```

```cpp
// ❌ SALAH — Blocking, freeze semua proses lain
void loop() {
    readSensors();
    delay(2000);         // ← DILARANG
    publishMQTT();
}
```

### 9.3 Pattern Wajib — GP2Y1010 Timing (Microseconds)

Sensor GP2Y1010 memerlukan timing 280µs yang sangat presisi. Gunakan `micros()` dengan state machine:

```cpp
// ✅ BENAR — State machine untuk timing GP2Y1010
typedef enum {
    DUST_IDLE,
    DUST_LED_ON,
    DUST_WAIT_SAMPLE,
    DUST_READING,
    DUST_LED_OFF
} DustState;

DustState dustState = DUST_IDLE;
unsigned long dustMicrosStart = 0;
float lastDustReading = 0.0f;

void dustSensorTick() {
    unsigned long now = micros();

    switch (dustState) {
        case DUST_IDLE:
            // Dipanggil dari timer interval 2000ms
            digitalWrite(PIN_DUST_LED, LOW);  // LED ON (active low)
            dustMicrosStart = now;
            dustState = DUST_WAIT_SAMPLE;
            break;

        case DUST_WAIT_SAMPLE:
            if (now - dustMicrosStart >= DUST_SAMPLE_DELAY_US) {
                // Baca ADC setelah 280µs
                int raw = analogRead(PIN_DUST_AOUT);
                float voltage = (raw / ADC_RESOLUTION) * ADC_VREF;
                voltage *= DUST_VDIVIDER_RATIO; // Koreksi voltage divider
                lastDustReading = voltageToDustDensity(voltage);
                dustState = DUST_LED_OFF;
            }
            break;

        case DUST_LED_OFF:
            if (now - dustMicrosStart >= DUST_LED_ON_US) {
                digitalWrite(PIN_DUST_LED, HIGH); // LED OFF
                dustState = DUST_IDLE;
            }
            break;
    }
}
```

### 9.4 MQTT Loop — Wajib Non-Blocking

```cpp
// ✅ BENAR
void loop() {
    mqttClient.loop();    // Panggil SETIAP loop() — proses incoming messages
    // ... timer checks ...
}

// ❌ SALAH — MQTT akan timeout jika terlalu jarang dipanggil
void loop() {
    delay(1000);
    mqttClient.loop();   // ← terlalu jarang, koneksi putus
}
```

### 9.5 WiFi Reconnect — Non-Blocking

```cpp
// ✅ BENAR — Non-blocking reconnect
void WiFiManager::loop() {
    unsigned long now = millis();
    if (WiFi.status() != WL_CONNECTED) {
        if (now - lastReconnectAttempt >= RECONNECT_INTERVAL_MS) {
            lastReconnectAttempt = now;
            WiFi.reconnect();
        }
    }
}

// ❌ SALAH — Blocking
void reconnectWiFi() {
    WiFi.begin(SSID, PASS);
    while (WiFi.status() != WL_CONNECTED) {   // ← BLOCKING LOOP
        delay(500);
    }
}
```

---

## 10. Do & Don't

### ✅ DO

| # | Harus Dilakukan | Alasan |
|---|----------------|--------|
| 1 | Gunakan **ADC1 saja** (GPIO32–GPIO39) | ADC2 tidak bisa dipakai saat WiFi aktif |
| 2 | Pasang **voltage divider** pada semua output sensor 5V | ESP32 ADC maksimal 3.3V — tanpa ini ADC/GPIO bisa rusak |
| 3 | Pasang **kapasitor filter** (220µF + 100nF) dekat GP2Y1010 | Spike arus saat LED aktif menyebabkan noise pembacaan |
| 4 | Panggil `mqttClient.loop()` **setiap iterasi** `loop()` | MQTT keepalive & proses incoming message |
| 5 | Gunakan **millis()/micros()** untuk semua timing | Non-blocking — sistem tetap responsif |
| 6 | Implementasikan **auto-reconnect** WiFi & MQTT | Koneksi bisa putus kapan saja |
| 7 | Publikasikan **LWT (Last Will Testament)** ke MQTT | Beri tahu broker jika device offline secara tiba-tiba |
| 8 | **Kalibrasi MQ135** di udara bersih sebelum deployment | Nilai Ro berbeda antar sensor |
| 9 | **Warm-up 20 menit** MQ135 sebelum mulai baca | Heater butuh waktu stabil |
| 10 | Simpan **secrets** (WiFi/MQTT creds) di `secrets.h` yang di-.gitignore | Keamanan kredensial |
| 11 | Pasang **pull-down 10kΩ** di Gate MOSFET fan | Pastikan fan OFF saat ESP32 boot/reset |
| 12 | Pasang **dioda flyback** paralel pada fan motor | Proteksi tegangan balik dari motor |
| 13 | Gunakan **PWM 25 kHz** untuk kontrol fan | Di atas frekuensi pendengaran manusia (inaudible) |
| 14 | Log semua event penting via Serial (debug mode) | Memudahkan troubleshooting |
| 15 | Validasi payload JSON sebelum publish MQTT | Cegah corrupt data di broker |

### ❌ DON'T

| # | Jangan Dilakukan | Akibat |
|---|-----------------|--------|
| 1 | **JANGAN** gunakan `delay()` di `loop()` | Memblokir MQTT loop, WiFi, & sensor timing — sistem freeze |
| 2 | **JANGAN** sambungkan output sensor 5V langsung ke GPIO ESP32 | GPIO ESP32 toleransi 3.3V — chip bisa rusak permanen |
| 3 | **JANGAN** gunakan ADC2 (GPIO 0,2,4,12-15,25-27) saat WiFi aktif | ADC2 conflict dengan WiFi — pembacaan error/crash |
| 4 | **JANGAN** hubungkan fan langsung ke GPIO ESP32 | Fan butuh arus tinggi (>12mA) — GPIO esp32 max 12mA, bisa rusak |
| 5 | **JANGAN** lupa panggil `mqttClient.loop()` | Koneksi MQTT putus, pesan masuk tidak diproses |
| 6 | **JANGAN** publish MQTT terlalu cepat (< 500ms interval) | Broker bisa throttle/disconnect, heap overflow |
| 7 | **JANGAN** hardcode WiFi/MQTT credentials di source code utama | Keamanan — jika repo publik, kredensial bocor |
| 8 | **JANGAN** abaikan warmup time MQ135 | Pembacaan tidak akurat, false positive alarm |
| 9 | **JANGAN** baca ADC GP2Y1010 tanpa LED trigger yang tepat | Tegangan output flat 0V — data tidak valid |
| 10 | **JANGAN** gunakan `String` class secara berulang dalam loop | Memory fragmentation — heap overflow / crash pada ESP32 |
| 11 | **JANGAN** gunakan `blocking reconnect loop` saat disconnect | Freeze semua fungsi lain termasuk sensor read |
| 12 | **JANGAN** lupa `mqttClient.setBufferSize()` untuk payload besar | Default buffer 256 bytes — JSON bisa terpotong |
| 13 | **JANGAN** jalankan fan tanpa proteksi dioda flyback | Spike tegangan induksi bisa merusak MOSFET / ESP32 |
| 14 | **JANGAN** biarkan ESP32 tanpa watchdog timer (WDT) | Loop hang tidak akan auto-recovery |
| 15 | **JANGAN** commit `secrets.h` ke version control | Kebocoran kredensial WiFi dan MQTT broker |

---

## 11. Error Handling

### 11.1 Error Code

```cpp
typedef enum {
    ERR_NONE            = 0,
    ERR_WIFI_TIMEOUT    = 1,   // WiFi gagal connect setelah N retry
    ERR_MQTT_TIMEOUT    = 2,   // MQTT broker tidak bisa dihubungi
    ERR_DUST_READ_FAIL  = 3,   // ADC membaca 0 terus-menerus
    ERR_GAS_WARMUP      = 4,   // MQ135 belum warm-up
    ERR_JSON_OVERFLOW   = 5,   // JSON buffer overflow
    ERR_FAN_FAULT       = 6,   // Fan tidak merespons (opsional, jika ada feedback)
} SystemError;
```

### 11.2 Error Response Policy

| Error | Response |
|-------|----------|
| WiFi disconnect | Auto-reconnect non-blocking, continue sensor read & store locally |
| MQTT disconnect | Auto-reconnect non-blocking, queue last N payload, publish saat reconnect |
| Dust sensor read 0V terus | Flag error, publish `null` untuk field dust |
| MQ135 warmup belum selesai | Publish data dengan flag `"warming_up": true` |
| JSON build gagal | Skip publish, log error, lanjut cycle berikutnya |

---

## 12. Parameter Konfigurasi

### 12.1 Ringkasan Parameter Tuning

| Parameter | Default | Range | Keterangan |
|-----------|---------|-------|------------|
| `SENSOR_READ_INTERVAL_MS` | 2000 | 500–60000 | Interval baca sensor |
| `DUST_SAMPLE_DELAY_US` | 280 | 270–300 | Delay setelah LED ON sebelum baca ADC |
| `DUST_LED_ON_US` | 320 | 310–350 | Total durasi LED aktif |
| `AQI_GOOD_MAX` | 35.4 | Sesuai standar | Threshold kategori Baik (µg/m³) |
| `AQI_MODERATE_MAX` | 75.4 | Sesuai standar | Threshold kategori Sedang |
| `FAN_SPEED_LOW` | 80/255 | 0–255 | PWM duty cycle fan speed rendah |
| `FAN_PWM_FREQ` | 25000 | 1000–40000 | Frekuensi PWM fan (Hz) |
| `MQTT_KEEPALIVE` | 60 | 10–120 | MQTT keepalive interval (detik) |
| `RECONNECT_INTERVAL_MS` | 5000 | 1000–30000 | Interval coba reconnect |

---

## Revisi Log

| Versi | Tanggal | Author | Perubahan |
|-------|---------|--------|-----------|
| 1.0.0 | 2025-01-01 | — | Initial release |

---

*FSD ini adalah dokumen hidup. Setiap perubahan hardware atau logika utama harus diperbarui di sini sebelum implementasi.*
