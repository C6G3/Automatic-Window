# Auto Window Dashboard

A full-stack IoT system for remotely monitoring and controlling an ESP32-driven motorised window. The dashboard runs as a Progressive Web App (PWA) and communicates with the ESP32 in real time over MQTT. Historical sensor data is persisted to Supabase and visualised on a live graph.

---

## Features

- **Real-time control** — set window angle, toggle auto/manual mode, adjust temperature threshold, and enable vent mode from any browser
- **Live sensor data** — temperature (DHT11), humidity, rain detection (water sensor), and motion (HW-416B PIR) streamed over MQTT
- **Historical graph** — time-series charts powered by Supabase (live, 15 m, 30 m, 1 h, 6 h, 1 d ranges)
- **PWA** — installable on Android/iOS, works offline for settings; manifest + service worker included
- **MQTT → Supabase bridge** — a lightweight Node.js ingest process (`ingest/bridge.mjs`) subscribes to all dashboard topics and writes readings + settings to two Supabase tables
- **Self-hosted or Vercel** — static frontend served by Express or deployed directly to Vercel; bridge runs alongside as a sidecar

---

## Architecture

```
Browser (PWA)
   │  WebSocket (MQTT over WSS)
   ▼
HiveMQ Public Broker  ◄────────────────►  ESP32 (PlatformIO)
   │                                         ├─ Servo motor (window)
   │  MQTT topics                            ├─ DHT11 (temp/humidity)
   ▼                                         ├─ Water sensor
Node.js Bridge (ingest/bridge.mjs)           └─ PIR motion sensor
   │
   ▼
Supabase (PostgreSQL)
   │
   └─► REST API ◄── app.js (historical data fetch)
```

### MQTT Topics

| Topic | Direction | Description |
|---|---|---|
| `home/dashboard/data` | ESP32 → Dashboard | Sensor telemetry (temp, humidity, motion, rain, angle) |
| `home/dashboard/settings` | Dashboard ↔ ESP32 | Settings payload (threshold, auto, vent, max_angle, …) |
| `home/dashboard/settings/get` | Dashboard → Bridge | Request latest settings snapshot |
| `home/dashboard/settings_snapshot` | Bridge → All | Retained snapshot of current settings |
| `home/dashboard/window/stream` | Dashboard → ESP32 | Live angle stream while dragging knob |
| `home/dashboard/angle_special` | ESP32 → Dashboard | Special angle overrides (e.g. rain-close) |
| `home/window/status` | ESP32 → Dashboard | Online/offline LWT (retained) |
| `home/window/heartbeat` | ESP32 → Dashboard | 2.5 s keepalive with RSSI + heap |

---

## Repository Layout

```
auto-window-dashboard/
├── index.html              # PWA shell
├── styles.css              # Dark/light theme styles
├── app.js                  # Frontend logic (MQTT, gauges, graph, settings)
├── config.js               # Generated at build — injects env vars into window.*
├── server.js               # Express server (serves static files + starts bridge)
├── manifest.webmanifest    # PWA manifest
├── vercel.json             # Vercel deployment config
│
├── ESP32/                  # PlatformIO firmware
│   ├── src/main.cpp        # Main firmware (servo, DHT11, PIR, water, MQTT)
│   ├── include/
│   │   ├── secrets.h       # ⚠ NOT in git — your Wi-Fi credentials go here
│   │   └── secrets.h.example  # Template to copy from
│   └── platformio.ini      # Board + library dependencies
│
├── ingest/
│   ├── bridge.mjs          # MQTT → Supabase bridge
│   ├── .env.example        # Template for bridge environment variables
│   └── seed_readings.mjs   # Dev utility: seed fake telemetry rows
│
├── supabase/
│   └── schema.sql          # Full DB schema (run once in Supabase SQL editor)
│
├── scripts/
│   └── gen-config.cjs      # Build script: writes config.js from .env
│
├── 3D Model/               # SolidWorks + STL files for housing (panel, base, PIR box)
├── docs/                   # Extra documentation and screenshots
└── icons/                  # PWA icons + wallpaper images
```

---

## Getting Started

### 1 — Supabase

1. Create a new Supabase project
2. Open the SQL editor and run `supabase/schema.sql`
3. Note your **Project URL** and **anon key** (for the frontend) and **service-role key** (for the bridge)

### 2 — Local development (web + bridge)

```bash
# Install dependencies
cd auto-window-dashboard
npm install

# Configure the frontend
cp .env.example .env.local
# Edit .env.local with your Supabase URL, anon key, and MQTT broker URL

# Generate config.js (reads .env.local)
node scripts/gen-config.cjs

# Start the server (also auto-starts the bridge via server.js)
npm start
# → http://localhost:3000
```

The bridge reads its own `.env` from `ingest/`:

```bash
cd ingest
cp .env.example .env
# Edit ingest/.env with your Supabase service-role key and MQTT URL
```

### 3 — Vercel deployment

```bash
vercel --prod
# Set all NEXT_PUBLIC_* environment variables in the Vercel dashboard
# The bridge must run elsewhere (Render, Railway, your Pi, etc.)
```

### 4 — ESP32 firmware

> **Prerequisites:** [PlatformIO](https://platformio.org/) (VS Code extension or CLI)

```bash
cd ESP32

# 1. Create your secrets file (never commit this)
cp include/secrets.h.example include/secrets.h

# 2. Edit secrets.h with your Wi-Fi credentials
#    #define WIFI_SSID "YourNetwork"
#    #define WIFI_PASS "YourPassword"

# 3. Build & flash
pio run --target upload

# 4. Monitor serial output
pio device monitor --baud 115200
```

#### Hardware wiring

| Component | ESP32 GPIO |
|---|---|
| Servo motor | 13 |
| DHT11 data | 4 |
| PIR (HW-416B) | 34 |
| Water sensor | 25 |
| LED 1 | 18 |
| LED 2 | 19 |
| LED 3 | 23 |
| Board LED | 2 |

#### PlatformIO dependencies

```ini
lib_deps =
    madhephaestus/ESP32Servo
    adafruit/DHT sensor library
    knolleary/PubSubClient
    bblanchon/ArduinoJson
```

---

## Environment Variables

### Frontend (`.env.local` / Vercel dashboard)

| Variable | Description |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase anon/public key |
| `NEXT_PUBLIC_MQTT_URL` | Broker WebSocket URL e.g. `wss://broker.hivemq.com:8884/mqtt` |
| `NEXT_PUBLIC_MQTT_USERNAME` | Optional broker username |
| `NEXT_PUBLIC_MQTT_PASSWORD` | Optional broker password |
| `NEXT_PUBLIC_MQTT_CLIENT_ID_PREFIX` | Client ID prefix (default `dashboard-`) |

### Bridge (`ingest/.env`)

| Variable | Description |
|---|---|
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_SERVICE_ROLE` | Service-role secret key |
| `MQTT_URL` | Broker WebSocket URL |
| `MQTT_USERNAME` / `MQTT_PASSWORD` | Optional broker credentials |
| `SUPABASE_READINGS` | Readings table name (default `readings`) |
| `SUPABASE_SETTINGS` | Settings table name (default `settings`) |
| `CLEANUP_ENABLED` | `true` to auto-purge old rows |
| `CLEANUP_PURGE_DAYS` | Rows older than N days are deleted (default `1`) |

---

## Security Notes

> **⚠ `ESP32/include/secrets.h` is listed in `.gitignore` and must never be committed.**
> It contains your Wi-Fi password in plaintext. Use `secrets.h.example` as the committed template.

> **The bridge uses the Supabase service-role key** which bypasses Row-Level Security. Keep it in `ingest/.env` only — this file is also excluded from git via `.gitignore`.

> **Public MQTT broker:** The default `broker.hivemq.com` is unauthenticated and shared. For production, use a private broker (HiveMQ Cloud, Mosquitto, etc.) and set `MQTT_USERNAME`/`MQTT_PASSWORD` in both the firmware and the bridge.

---

## 3D Printed Parts

The `3D Model/` folder contains SolidWorks (`.SLDPRT`) and exported STL files for:

- **`base.STL`** — main mounting bracket
- **`panel.STL`** — front panel / window attachment
- **`pir_box.STL`** — PIR sensor enclosure

Print in PLA at 0.2 mm layer height, 20 % infill.

---

## License

MIT — see [LICENSE](LICENSE).
