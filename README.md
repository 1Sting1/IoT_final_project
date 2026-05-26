# IoT Water Tank Automatic Filling and Drainage Control System

An intelligent distributed IoT system for remote monitoring and automatic control of water tank levels with leak detection, auto-fill capabilities, and multi-station cloud management.

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Hardware Components](#hardware-components)
- [Wiring Diagrams](#wiring-diagrams)
- [Software Stack](#software-stack)
- [Installation & Deployment](#installation--deployment)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Usage Guide](#usage-guide)
- [Troubleshooting](#troubleshooting)

---

## Overview

This project implements a complete IoT solution for automated water tank management:

- **Real-time monitoring** of water level and soil moisture
- **Automatic pump control** with hysteresis-based filling
- **Leak detection** algorithm to prevent water loss
- **Local control** via Electron desktop application on Raspberry Pi
- **Remote access** through cloud VPS with web interface
- **Multi-station support** for managing multiple tanks from one dashboard

### Key Features

| Feature | Description |
|---------|-------------|
| Level Monitoring | HC-SR04 ultrasonic sensor measures water level with ±1cm accuracy |
| Moisture Sensing | Capacitive soil moisture sensor for irrigation feedback |
| Auto-Fill Mode | Maintains target level with ±5% hysteresis to prevent pump cycling |
| Leak Detection | Detects unexpected water loss when soil is dry |
| Dual Interface | Local Electron app + Remote web dashboard |
| Data Persistence | PostgreSQL stores 7 days of measurements |
| Secure Remote Access | JWT authentication + HTTPS via Let's Encrypt |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLOUD (VPS)                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                      │
│  │   Traefik   │───▶│    Nginx    │───▶│  FastAPI    │                      │
│  │   (HTTPS)   │    │   (Proxy)   │    │  (Backend)  │                      │
│  └─────────────┘    └─────────────┘    └──────┬──────┘                      │
│                                               │                              │
│                     ┌─────────────┐           │ WebSocket                    │
│                     │  React Web  │◀──────────┘                              │
│                     │     UI      │                                          │
│                     └─────────────┘                                          │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ WebSocket (wss://)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LOCAL (Raspberry Pi)                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                      │
│  │ PostgreSQL  │◀───│  FastAPI    │───▶│ VPS Bridge  │                      │
│  │     DB      │    │  (Backend)  │    │  (Service)  │                      │
│  └─────────────┘    └──────┬──────┘    └─────────────┘                      │
│                            │                                                 │
│        ┌───────────────────┼───────────────────┐                            │
│        │                   │                   │                            │
│        ▼                   ▼                   ▼                            │
│  ┌───────────┐    ┌───────────────┐    ┌───────────────┐                    │
│  │ Electron  │    │ Pump Control  │    │ Leak Detector │                    │
│  │    UI     │    │   Service     │    │   Service     │                    │
│  └───────────┘    └───────────────┘    └───────────────┘                    │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │ WebSocket (ws://)
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HARDWARE (Arduino)                                   │
│                                                                              │
│  ┌─────────────────────────┐        ┌─────────────────────────┐             │
│  │      NANO #1            │        │      NANO #2            │             │
│  │  ┌─────────┐ ┌───────┐  │        │  ┌─────────┐            │             │
│  │  │ HC-SR04 │ │ Relay │  │        │  │ YL-69   │            │             │
│  │  │ (Level) │ │ x2    │  │        │  │(Moisture│            │             │
│  │  └────┬────┘ └───┬───┘  │        │  └────┬────┘            │             │
│  │       │          │      │        │       │                 │             │
│  │  ┌────┴──────────┴───┐  │        │  ┌────┴──────────────┐  │             │
│  │  │   Arduino Nano    │  │        │  │   Arduino Nano    │  │             │
│  │  └─────────┬─────────┘  │        │  └─────────┬─────────┘  │             │
│  │            │            │        │            │            │             │
│  │  ┌─────────┴─────────┐  │        │  ┌─────────┴─────────┐  │             │
│  │  │     ESP8266       │  │        │  │     ESP8266       │  │             │
│  │  │     (WiFi)        │  │        │  │     (WiFi)        │  │             │
│  │  └───────────────────┘  │        │  └───────────────────┘  │             │
│  └─────────────────────────┘        └─────────────────────────┘             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Sensor → Arduino**: Nano reads HC-SR04 (level) and YL-69 (moisture) sensors
2. **Arduino → ESP8266**: Serial communication at 9600 baud with JSON messages
3. **ESP8266 → Raspberry Pi**: WebSocket connection to local API
4. **Raspberry Pi → VPS**: WebSocket bridge for remote access
5. **VPS → Browser**: Real-time updates via WebSocket

### Command Flow

1. **User action** in UI (local Electron or remote web)
2. **API receives** command (REST or WebSocket)
3. **PumpController** validates and forwards to Arduino
4. **Arduino executes** relay control
5. **State update** propagates back through the chain

---

## Hardware Components

### Bill of Materials

| Component              | Quantity | Purpose |
|------------------------|----------|---------|
| Arduino Nano           | 2 | Microcontrollers for sensors and relay control |
| ESP8266 (ESP-01)       | 2 | WiFi modules for wireless communication |
| HC-SR04                | 1 | Ultrasonic distance sensor for water level |
| YL-69/LM393            | 1 | Capacitive soil moisture sensor |
| 2-Channel Relay Module | 1 | Pump control (5V coil, 10A contacts) |
| Raspberry Pi 5         | 1 | Local server and UI host |
| Water Pump             | 1-2 | Submersible or inline pump (12V/24V) |
| Power Supply 5V        | 2 | For Arduino and ESP8266 |
| Power Supply 12V/24V   | 1 | For pump (match pump voltage) |
| Jumper Wires           | - | Connections |
| Water Tank             | 1 | Rectangular tank (any size with calibration) |

### Pin Configuration

#### Nano #1 (Pump Control & Level Sensor)

| Pin | Connection | Description |
|-----|------------|-------------|
| D2 | ESP8266 TX | Software Serial RX |
| D3 | ESP8266 RX | Software Serial TX |
| D5 | Relay IN1 | Pump 1 control |
| D6 | Relay IN2 | Pump 2 control |
| D9 | HC-SR04 TRIG | Ultrasonic trigger |
| D10 | HC-SR04 ECHO | Ultrasonic echo |
| 5V | VCC | Power for sensors |
| GND | GND | Common ground |

#### Nano #2 (Moisture Sensor)

| Pin | Connection | Description |
|-----|------------|-------------|
| A0 | YL-69 Signal | Analog moisture reading |
| D2 | ESP8266 TX | Software Serial RX |
| D3 | ESP8266 RX | Software Serial TX |
| 5V | VCC | Power for sensor |
| GND | GND | Common ground |

---

## Wiring Diagrams

### Nano #1 - Level Sensor & Pump Control

```
                                    ┌─────────────────┐
                                    │   HC-SR04       │
                                    │  ┌───┬───┬───┐  │
                                    │  │VCC│TRG│ECH│GND│
                                    │  └─┬─┴─┬─┴─┬─┴─┬─┘
                                    └────│───│───│───│──┘
                                         │   │   │   │
    ┌────────────────────────────────────│───│───│───│────────────────┐
    │                                    │   │   │   │                │
    │  ┌──────────────────────────────┐  │   │   │   │                │
    │  │      ARDUINO NANO            │  │   │   │   │                │
    │  │                              │  │   │   │   │                │
    │  │  5V ─────────────────────────┼──┘   │   │   │                │
    │  │  GND ────────────────────────┼──────│───│───┘                │
    │  │  D9 (TRIG) ──────────────────┼──────┘   │                    │
    │  │  D10 (ECHO) ─────────────────┼──────────┘                    │
    │  │                              │                               │
    │  │  D5 (RELAY1) ────────────────┼──────────────┐                │
    │  │  D6 (RELAY2) ────────────────┼────────────┐ │                │
    │  │                              │            │ │                │
    │  │  D2 (RX) ◄───────────────────┼────────┐   │ │                │
    │  │  D3 (TX) ────────────────────┼──────┐ │   │ │                │
    │  │                              │      │ │   │ │                │
    │  └──────────────────────────────┘      │ │   │ │                │
    │                                        │ │   │ │                │
    │  ┌──────────────────────────────┐      │ │   │ │                │
    │  │      ESP8266 (ESP-01)        │      │ │   │ │                │
    │  │                              │      │ │   │ │                │
    │  │  TX ─────────────────────────┼──────┘ │   │ │                │
    │  │  RX ◄────────────────────────┼────────┘   │ │                │
    │  │  VCC ── 3.3V (!)             │            │ │                │
    │  │  GND ── GND                  │            │ │                │
    │  │  CH_PD ── 3.3V               │            │ │                │
    │  └──────────────────────────────┘            │ │                │
    │                                              │ │                │
    │  ┌──────────────────────────────┐            │ │                │
    │  │    2-CHANNEL RELAY MODULE    │            │ │                │
    │  │                              │            │ │                │
    │  │  IN1 ◄───────────────────────┼────────────│─┘                │
    │  │  IN2 ◄───────────────────────┼────────────┘                  │
    │  │  VCC ── 5V                   │                               │
    │  │  GND ── GND                  │                               │
    │  │                              │      ┌─────────────┐          │
    │  │  COM1 ────────────────────────────── │   PUMP 1   │          │
    │  │  NO1 ─────────────────────────────── │   12V/24V  │          │
    │  │                              │      └─────────────┘          │
    │  │  COM2 ────────────────────────────── ┌─────────────┐         │
    │  │  NO2 ─────────────────────────────── │   PUMP 2   │          │
    │  │                              │      └─────────────┘          │
    │  └──────────────────────────────┘                               │
    └─────────────────────────────────────────────────────────────────┘
```

### Nano #2 - Moisture Sensor

```
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  ┌──────────────────────────────┐      ┌──────────────────┐    │
    │  │      ARDUINO NANO            │      │  YL-69 MOISTURE  │    │
    │  │                              │      │     SENSOR       │    │
    │  │  A0 ◄────────────────────────┼──────┤ AO (Analog Out)  │    │
    │  │  5V ─────────────────────────┼──────┤ VCC              │    │
    │  │  GND ────────────────────────┼──────┤ GND              │    │
    │  │                              │      └──────────────────┘    │
    │  │  D2 (RX) ◄───────────────────┼────────┐                     │
    │  │  D3 (TX) ────────────────────┼──────┐ │                     │
    │  │                              │      │ │                     │
    │  └──────────────────────────────┘      │ │                     │
    │                                        │ │                     │
    │  ┌──────────────────────────────┐      │ │                     │
    │  │      ESP8266 (ESP-01)        │      │ │                     │
    │  │                              │      │ │                     │
    │  │  TX ─────────────────────────┼──────┘ │                     │
    │  │  RX ◄────────────────────────┼────────┘                     │
    │  │  VCC ── 3.3V (!)             │                              │
    │  │  GND ── GND                  │                              │
    │  │  CH_PD ── 3.3V               │                              │
    │  └──────────────────────────────┘                              │
    │                                                                 │
    └─────────────────────────────────────────────────────────────────┘
```

### Power Supply Connections

```
    ┌───────────────────────────────────────────────────────────────┐
    │                    POWER DISTRIBUTION                         │
    │                                                               │
    │   ┌─────────────┐                                             │
    │   │  5V PSU     │                                             │
    │   │  (2A min)   │                                             │
    │   └──────┬──────┘                                             │
    │          │                                                    │
    │          ├────────────► Arduino Nano #1 (VIN or 5V)           │
    │          ├────────────► Arduino Nano #2 (VIN or 5V)           │
    │          ├────────────► Relay Module VCC                      │
    │          │                                                    │
    │          │  ┌───────────────────────┐                         │
    │          └──┤ 3.3V Voltage Regulator├──► ESP8266 x2 VCC       │
    │             │    (AMS1117-3.3V)     │                         │
    │             └───────────────────────┘                         │
    │                                                               │
    │   ┌─────────────┐                                             │
    │   │ 12V/24V PSU │                                             │
    │   │ (pump rated)│                                             │
    │   └──────┬──────┘                                             │
    │          │                                                    │
    │          └────────────► Relay COM (through NO contacts)       │
    │                         ► Pump power supply                   │
    │                                                               │
    │   NOTE: ESP8266 requires 3.3V! Do not connect to 5V!          │
    │   Use AMS1117-3.3V regulator or dedicated 3.3V supply.        │
    │                                                               │
    └───────────────────────────────────────────────────────────────┘
```

### Tank Sensor Placement

```
    ┌─────────────────────────────────────────┐
    │              HC-SR04 SENSOR             │  ◄── Mounted on top
    │                   │                     │      facing down
    │                   ▼                     │
    ├─────────────────────────────────────────┤  ◄── distance_full
    │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │      (tank full)
    │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
    │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
    │ ░░░░░░░░░░░░░ WATER ░░░░░░░░░░░░░░░░░░░ │
    │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
    │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
    ├─────────────────────────────────────────┤  ◄── Current level
    │                                         │      (measured distance)
    │                                         │
    │                                         │
    │                                         │
    │                                         │
    ├─────────────────────────────────────────┤  ◄── distance_empty
    │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│      (tank empty)
    └─────────────────────────────────────────┘

    Level Calculation:
    level_pct = (distance_empty - measured) / (distance_empty - distance_full) × 100%
```

---

## Software Stack

### Backend Technologies

| Component | Technology | Version |
|-----------|------------|---------|
| Raspberry Pi API | FastAPI | 0.115.0 |
| VPS API | FastAPI | 0.115.0 |
| Database | PostgreSQL | 16 |
| ORM | SQLAlchemy | 2.0.36 |
| Async DB Driver | asyncpg | 0.30.0 |
| WebSocket | websockets | 13.1 |
| Authentication | python-jose (JWT) | 3.3.0 |
| Password Hashing | bcrypt | 4.2.1 |

### Frontend Technologies

| Component | Technology | Version |
|-----------|------------|---------|
| Local UI | Electron | 33.2.1 |
| Web UI | React | 18.3.1 |
| Routing | React Router | 6.28.0 |
| Charts | Recharts | 2.13.3 |
| HTTP Client | Axios | 1.7.9 |
| Build Tool (Local) | esbuild | 0.28.0 |
| Build Tool (Web) | Vite | 6.0.3 |

### Infrastructure

| Component | Technology | Version |
|-----------|------------|---------|
| Reverse Proxy | Traefik | 3.2 |
| Web Server | Nginx | 1.27 |
| Containerization | Docker | - |
| Orchestration | Docker Compose | - |
| TLS Certificates | Let's Encrypt | - |

---

## Installation & Deployment

### Prerequisites

- Raspberry Pi 5 with Raspberry Pi OS
- Docker and Docker Compose installed
- Arduino IDE for flashing microcontrollers
- Node.js 18+ (for building Electron app)
- VPS with public IP (for remote access)

### 1. Arduino Setup

#### Flash Nano #1 (Pump Control)

```bash
# Open Arduino IDE
# File → Open → arduino/nano1_pumps/nano1_pumps.ino

# Configure WiFi credentials in ESP8266 code:
# - WIFI_SSID
# - WIFI_PASSWORD
# - WEBSOCKET_HOST (Raspberry Pi IP)
# - WEBSOCKET_PORT (8000)

# Select Board: Arduino Nano
# Select Processor: ATmega328P (Old Bootloader)
# Upload
```

#### Flash Nano #2 (Moisture Sensor)

```bash
# Open Arduino IDE
# File → Open → arduino/nano2_moisture/nano2_moisture.ino

# Same WiFi configuration as Nano #1
# Upload
```

### 2. Raspberry Pi Deployment

```bash
# Clone repository
git clone <repository-url>
cd IoT_final_project/raspberry

# Create environment file
cp .env.example .env

# Edit configuration
nano .env
```

**.env configuration:**
```env
DATABASE_URL=postgresql+asyncpg://water:secret@db:5432/waterdb
VPS_WS_URL=wss://your-domain.com/ws/raspberry
VPS_API_KEY=your-secure-api-key
STATION_ID=station_001
POSTGRES_DB=waterdb
POSTGRES_USER=water
POSTGRES_PASSWORD=secret
```

```bash
# Start services
docker-compose up -d

# Check logs
docker-compose logs -f api
```

#### Build Electron UI

```bash
cd electron-ui

# Install dependencies
npm install

# Build
npm run build

# Run
npm start
```

### 3. VPS Deployment

```bash
# On your VPS
git clone <repository-url>
cd IoT_final_project/vps

# Create environment file
cp .env.example .env

# Generate password hash
python3 -c "import bcrypt; print(bcrypt.hashpw(b'your-password', bcrypt.gensalt()).decode())"

# Edit configuration
nano .env
```

**.env configuration:**
```env
API_KEY=your-secure-api-key
ADMIN_USERNAME=admin
ADMIN_PASSWORD_HASH=$2b$12$...  # bcrypt hash from above
JWT_SECRET=your-jwt-secret-key
RASPBERRY_API_URL=http://192.168.1.100:8000
DOMAIN=your-domain.com
```

```bash
# Create external network for Traefik
docker network create proxy

# Start services
docker-compose up -d

# Check logs
docker-compose logs -f
```

### 4. DNS Configuration

Point your domain to your VPS IP address:
```
A    your-domain.com    → VPS_IP
```

Traefik will automatically obtain SSL certificates from Let's Encrypt.

---

## Configuration

### Tank Calibration

Configure tank dimensions and sensor calibration via API or UI:

```json
{
  "calibration": {
    "length": 100,          // Tank length in cm
    "width": 50,            // Tank width in cm
    "height": 60,           // Tank height in cm
    "distance_empty": 65,   // Distance reading when tank is empty (cm)
    "distance_full": 5      // Distance reading when tank is full (cm)
  }
}
```

### Moisture Sensor Calibration

Edit constants in `arduino/nano2_moisture/nano2_moisture.ino`:

```cpp
const int DRY_VALUE = 800;   // Analog reading in dry air
const int WET_VALUE = 300;   // Analog reading in water
```

### Auto-Mode Settings

- **Target Level**: Set via UI (0-100%)
- **Hysteresis**: ±5% (hardcoded in `auto_mode.py`)
- **Behavior**: Pumps ON when level < target-5%, OFF when level > target+5%

### Leak Detection Thresholds

Edit in `raspberry/api/services/leak_detection.py`:

```python
LEAK_THRESHOLD_PCT_PER_SEC = 0.05  # 5%/minute drop rate
MOISTURE_DRY_THRESHOLD = 30.0      # Below this = dry soil
```

---

## API Reference

### Raspberry Pi API (Local)

Base URL: `http://localhost:8000`

#### Stations

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/stations` | List all stations |
| POST | `/api/stations` | Create station |
| GET | `/api/stations/{id}` | Get station details |
| PATCH | `/api/stations/{id}` | Update station |
| DELETE | `/api/stations/{id}` | Delete station |

#### Commands

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| POST | `/api/stations/{id}/pumps` | `{"action": "on"\|"off"}` | Control pumps |
| POST | `/api/stations/{id}/mode` | `{"mode": "auto"\|"manual", "target_level": 80}` | Set mode |
| GET | `/api/stations/{id}/events` | - | Get event history |

#### Measurements

| Method | Endpoint | Parameters | Description |
|--------|----------|------------|-------------|
| GET | `/api/stations/{id}/measurements` | `from`, `to`, `limit` | Query history |

#### WebSocket

| Endpoint | Description |
|----------|-------------|
| `/ws/arduino` | Arduino sensor data ingestion |
| `/ws/clients` | Client real-time updates |

### VPS API (Remote)

Base URL: `https://your-domain.com`

#### Authentication

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| POST | `/auth/login` | `{"username": "...", "password": "..."}` | Get JWT token |
| GET | `/auth/me` | - | Get current user (requires Bearer token) |

#### Stations

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/stations` | List online stations |
| GET | `/api/stations/{id}` | Get station state |
| POST | `/api/stations/{id}/commands` | Send command |
| GET | `/api/stations/{id}/measurements` | Proxy to Raspberry Pi |
| GET | `/api/stations/{id}/events` | Proxy to Raspberry Pi |

#### WebSocket

| Endpoint | Description |
|----------|-------------|
| `/ws/raspberry?x-api-key=...` | Raspberry Pi connection |
| `/ws/browser?token=...` | Browser client connection |

---

## Usage Guide

### Local Control (Electron UI)

1. **Dashboard**: View real-time level, moisture, and pump status
2. **Pump Control**: Click ON/OFF buttons (blocked when tank full)
3. **Auto Mode**: Enable and set target level percentage
4. **History**: View charts of historical data
5. **Calibration**: Set tank dimensions and sensor calibration

### Remote Control (Web UI)

1. **Login**: Enter username and password
2. **Stations**: View all online stations
3. **Station Detail**: Full control interface for selected station
4. **History**: Query and visualize historical data

### LED/Relay Indicators

| State | Relay LED | Meaning |
|-------|-----------|---------|
| Pumps OFF | OFF | No pumping |
| Pumps ON | ON | Active pumping |
| Leak Detected | Flashing (software) | Check for leaks |

---

## Troubleshooting

### Arduino Not Connecting

1. **Check WiFi credentials** in ESP8266 code
2. **Verify Raspberry Pi IP** is correct
3. **Check serial connection** between Nano and ESP8266
4. **Monitor serial output** at 115200 baud

### No Sensor Data

1. **Check wiring** according to diagrams
2. **Verify sensor power** (5V for HC-SR04, 5V for YL-69)
3. **Test sensors independently** with simple sketches

### WebSocket Connection Failed

1. **Check firewall** allows port 8000
2. **Verify Docker containers** are running
3. **Check API logs**: `docker-compose logs api`

### VPS Not Receiving Data

1. **Verify VPS_WS_URL** in Raspberry Pi config
2. **Check API_KEY** matches on both sides
3. **Verify TLS certificates** are valid
4. **Check Traefik logs**: `docker-compose logs traefik`

### Database Issues

```bash
# Reset database
docker-compose down -v
docker-compose up -d

# Check database logs
docker-compose logs db
```

### Pump Not Responding

1. **Check relay wiring** and power
2. **Verify Arduino received command** via serial monitor
3. **Test relay manually** with direct signal
4. **Check pump power supply** voltage

---

## Project Structure

```
IoT_final_project/
├── arduino/
│   ├── nano1_pumps/           # Level sensor & pump control
│   └── nano2_moisture/        # Moisture sensor
├── raspberry/
│   ├── api/                   # FastAPI backend
│   │   ├── models/            # Database & Pydantic schemas
│   │   ├── routers/           # API endpoints
│   │   └── services/          # Business logic
│   ├── electron-ui/           # Desktop application
│   └── docker-compose.yml     # Local deployment
├── vps/
│   ├── api/                   # Cloud backend
│   │   └── routers/           # API endpoints
│   ├── web/                   # React web UI
│   ├── nginx/                 # Reverse proxy config
│   ├── traefik/               # TLS termination
│   └── docker-compose.yml     # Cloud deployment
└── README.md                  # This file
```

---

## License



## Authors

[Kotelnikov Maksim](https://github.com/1Sting1/)

[Efimov Konstantin](https://github.com/Krokodilchik234/)

[Zalygin Aleksandr](https://github.com/AleksandrZalygin/)
