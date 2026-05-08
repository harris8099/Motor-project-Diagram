# ESP32-Based Smart Motor Retrofit and Monitoring System
### Final Year Project — Technical Report

---

## 1. System Overview

This project implements a full-stack IoT system for **real-time motor health monitoring and predictive maintenance**. A physical ESP32 microcontroller collects electrical and mechanical sensor data from a motor, relays it over Wi-Fi to a cloud backend, stores it in a managed PostgreSQL database (Neon), and exposes it through a React dashboard hosted on Vercel. Machine learning models running on the backend continuously predict faults, degradation, and remaining useful life.

---

## 2. High-Level System Architecture

```mermaid
graph TB
    subgraph EDGE["⚙️ Edge Layer — ESP32 (ESP-IDF / FreeRTOS)"]
        PZEM["PZEM-004T\nVoltage · Current · Power\nEnergy · Frequency · PF"]
        DS18["DS18B20\nMotor Temperature (°C)"]
        ADXL["ADXL345\n3-Axis Vibration\nTap Detection"]
        HALL["Hall Sensor\nRPM Measurement"]
        RELAY["Relay Module\nMotor ON/OFF"]
        BUZZER["Buzzer\nFault Alert"]
        TFT["TFT Display\nLocal Status"]
        ESP["ESP32-WROOM-32\nFreeRTOS Kernel"]
        PZEM --> ESP
        DS18 --> ESP
        ADXL --> ESP
        HALL --> ESP
        ESP --> RELAY
        ESP --> BUZZER
        ESP --> TFT
    end

    subgraph CLOUD["☁️ Cloud Layer — Render.com"]
        API["FastAPI Backend\nuvicorn · asyncpg"]
        WORKER["Redis Worker\nBackground Jobs"]
        ML["ML Pipeline\n5 Prediction Tiers"]
        API --> ML
        API --> WORKER
    end

    subgraph DB["🗄️ Database — Neon (PostgreSQL)"]
        TABLES["sensor_readings\npredictions\nfault_logs\ndevices\ncommands\nusers"]
    end

    subgraph FRONTEND["🖥️ Frontend — Vercel"]
        REACT["React 19 + Vite 8\nDashboard / Charts / AI Page"]
    end

    subgraph AI["🤖 AI Providers"]
        MISTRAL["Mistral AI\nmistral-small"]
        GEMINI["Google Gemini\ngemini-2.0-flash"]
        GROQ["Groq\nllama-3.3-70b-versatile"]
    end

    ESP --"HTTPS POST /data\n(Vercel Relay)"--> API
    ESP --"HTTPS POST /data\n(Railway Relay)"--> API
    ESP --"MQTT Publish"--> API
    API --"asyncpg SSL"--> TABLES
    REACT --"REST API + x-api-key"--> API
    ML --"Mistral → Gemini → Groq\nFallback Chain"--> AI
    TABLES --> ML
```

---

## 3. Data Flow: ESP32 → Render → Neon → Vercel

```mermaid
sequenceDiagram
    participant ESP as ESP32 (Edge)
    participant RND as Render.com (FastAPI)
    participant NEO as Neon (PostgreSQL)
    participant WKR as Redis Worker
    participant VRC as Vercel (React)
    participant OMA as Open-Meteo API

    Note over ESP: Every 5 seconds
    ESP->>RND: POST /data {SensorPayload JSON}\n+ x-api-key header

    RND->>NEO: SELECT last fault state
    NEO-->>RND: previous fault row

    RND->>OMA: GET ambient temp (lat/lon)\n[if t2 == 0]
    OMA-->>RND: temperature_2m

    RND->>NEO: INSERT sensor_readings (30+ cols)
    RND->>NEO: UPSERT devices (heartbeat)
    RND->>NEO: INSERT fault_logs (new faults only)
    RND->>NEO: UPDATE fault_logs (cleared faults)

    RND->>WKR: enqueue_prediction_job(payload)\n[background task or Redis]
    RND-->>ESP: {ok: true}

    Note over WKR: Async — non-blocking
    WKR->>NEO: SELECT historical 72h data
    NEO-->>WKR: up to 200 readings
    WKR->>WKR: Run ML Pipeline\n(Isolation Forest, RF, NASA BHI, RUL...)
    WKR->>NEO: INSERT predictions (per model)
    WKR->>NEO: DELETE predictions > 7 days

    Note over VRC: User opens dashboard
    VRC->>RND: GET /data/{device_id}
    RND->>NEO: SELECT sensor_readings
    NEO-->>RND: rows
    RND-->>VRC: JSON response

    VRC->>RND: GET /predictions/{device_id}
    RND->>NEO: SELECT predictions
    NEO-->>RND: ML results
    RND-->>VRC: predictions JSON

    Note over VRC: User clicks "Analyze"
    VRC->>RND: POST /ai/analyze/{device_id}
    RND->>RND: Mistral → Gemini → Groq fallback
    RND->>NEO: INSERT ai_comment prediction
    RND-->>VRC: {analysis, comment}
```

---

## 4. Hardware Block Diagram

```mermaid
graph LR
    subgraph PWR["Power Supply"]
        MAINS["230V AC Mains"]
    end

    subgraph MOTOR["Induction Motor"]
        MTR["Single-Phase Motor\n~230V AC"]
    end

    subgraph PROT["Protection & Control"]
        RLY["Relay Module\nGPIO26 — Active Low\nON: Motor Energised"]
        BUZ["Buzzer\nGPIO27\nFault Alert"]
    end

    subgraph SENSORS["Sensor Suite"]
        PZ["PZEM-004T\nUART2 (GPIO17 TX, GPIO16 RX)\nVoltage · Current · Power\nEnergy · Frequency · PF"]
        DS["DS18B20\n1-Wire Protocol\nMotor Temperature Sensor"]
        AX["ADXL345\nI2C (SDA/SCL)\n±16g, 3-Axis Accel\nTap Detection"]
        HS["Hall Effect Sensor\nGPIO Interrupt\nPulse Count → RPM"]
    end

    subgraph ESP32["ESP32-WROOM-32 MCU"]
        CPU["Dual-Core Xtensa LX6\n240 MHz · 4MB Flash · 520KB SRAM"]
        NVS["NVS Flash Storage\nConfig + Fault State\n+ Uptime Persistence"]
        WIFI["Wi-Fi 802.11 b/g/n\n2.4 GHz"]
    end

    subgraph DISPLAY["Local UI"]
        TFT["TFT LCD Display\nSPI Interface\nReal-time Status"]
    end

    MAINS --> RLY
    RLY --> MTR
    MTR --> PZ
    PZ -->|UART| ESP32
    DS -->|1-Wire| ESP32
    AX -->|I2C| ESP32
    HS -->|GPIO IRQ| ESP32
    ESP32 --> RLY
    ESP32 --> BUZ
    ESP32 --> TFT
    ESP32 -->|HTTPS / MQTT| CLOUD["☁️ Cloud"]
```

---

## 5. ESP32 Firmware Architecture (FreeRTOS Tasks)

```mermaid
graph TB
    subgraph BOOT["app_main() — Boot Sequence"]
        INIT["nvs_flash_init()\nconsole_log_start()"]
    end

    subgraph TASKS["FreeRTOS Concurrent Tasks"]
        T1["sensor_data()\nPriority 4 · 8192B stack\nReads all sensors every loop\nRuns protection engine"]
        T2["http_server\nPriority 3\nServes local web UI\nHandles /api/* endpoints"]
        T3["vercel_relay\nPriority 2\nHTTPS POST every 5s\nto Render backend"]
        T4["railway_relay\nPriority 2\nHTTPS POST alternate\nbackend"]
        T5["mqtt_relay\nPriority 2\nMQTT publish\nto broker"]
        T6["tft_display\nPriority 3\nUpdates LCD every 1s"]
        T7["wifi_app\nPriority 1\nManages Wi-Fi\nconnection"]
        PZEM_T["pzemTask()\nSeparate UART task\n300ms timeout isolated"]
    end

    subgraph SHARED["Shared State (Mutex Protected)"]
        GDATA["g_sensorData\nSensorData struct\nprotected by sensor_mutex"]
        PSTATE["sensor_state_lock\nportMUX spinlock\nfor fault + relay state"]
    end

    INIT --> TASKS
    T1 <--> GDATA
    T1 <--> PSTATE
    T3 --> GDATA
    T4 --> GDATA
    T5 --> GDATA
    T6 --> GDATA
    PZEM_T --> GDATA
```

---

## 6. Motor Protection Signal Flow

```mermaid
flowchart TD
    READ["Read Sensor Data\n(every task loop)"]
    GRACE{"Startup Grace\nPeriod Active?"}
    SUPPRESS["Suppress: overcurrent\nstall · vibration"]
    CHECK["Evaluate Raw Fault Conditions"]

    OC{"Current >\nmaxCurrentA?"}
    OT{"Temp1 > maxTempC\n3 consecutive samples?"}
    ST{"RPM < minRpm AND\nCurrent > stallCurrentA?"}
    VIB{"VibrationLevel ≥ threshold\nfor trip_duration ms?"}
    OV{"Voltage > overvoltageV\n+ 2.5V hysteresis?"}
    UV{"Voltage < undervoltageV\n- 2.5V hysteresis?"}

    LATCH["Latch Fault Flags\n(portMUX critical section)"]
    NVS_SAVE["Persist to NVS Flash\n(survives reboot)"]
    COUNT{"fault_count ≥\nfaultTripCount?"}
    TRIP["Trip: Relay OFF\nBuzzer ON\nMotor Stopped"]
    CONT["Continue Running\nUpdate motor_running flag"]

    READ --> GRACE
    GRACE -->|Yes| SUPPRESS
    GRACE -->|No| CHECK
    SUPPRESS --> CHECK
    CHECK --> OC & OT & ST & VIB & OV & UV
    OC & OT & ST & VIB & OV & UV --> LATCH
    LATCH --> NVS_SAVE
    NVS_SAVE --> COUNT
    COUNT -->|Yes| TRIP
    COUNT -->|No| CONT
```

---

## 7. Cloud Relay Communication Architecture

```mermaid
graph LR
    ESP["ESP32"]

    subgraph RELAYS["Data Relay Channels (configurable, stored in NVS)"]
        VR["Vercel Relay\nHTTPS POST\nintervalSeconds=5\nAPI Key auth"]
        RR["Railway Relay\nHTTPS POST\nintervalSeconds=5\nAPI Key auth"]
        MQ["MQTT Relay\nMQTT Publish\nQoS configurable\nBroker URI + credentials"]
    end

    subgraph BACKEND["Render.com — FastAPI"]
        EP["/data endpoint\nPOST"]
    end

    subgraph CMDFLOW["Bidirectional Command Flow"]
        POLL["ESP32 polls\nGET /command/{id}/pending\nevery 5s"]
        QUEUE["commands table\nPendng → Acked\nAuto-expire 5min"]
        DASH["Dashboard\nQueues commands:\nack_faults · set_protection\nset_maintenance · reset_energy"]
    end

    ESP --> VR & RR & MQ
    VR & RR --> EP
    MQ --> EP
    DASH --> QUEUE
    POLL --> QUEUE
    QUEUE --> ESP
```

---

## 8. Backend API Architecture

```mermaid
graph TB
    subgraph FAST["FastAPI Application (app/main.py)"]
        MW["CORS Middleware\nJWT Auth Middleware"]
        subgraph ROUTERS["API Routers"]
            DR["/data — data.py\nPOST /data · GET /data/{id}\n/ai/analyze · /command/*\n/ai/local-analyze"]
            FR["/faults — faults.py\nFault lifecycle management\ncreate · resolve · delete"]
            PR["/predictions — predictions.py\nGET stored ML results"]
            FCR["/forecast — forecast.py\nLinear regression\npower/energy/cost forecast"]
            DV["/devices — devices.py\nDevice CRUD\nbulk registration"]
            AU["/auth — auth.py\nJWT login · token verify"]
        end
    end

    subgraph SVC["Services Layer"]
        PB["prediction.py\nRule-based (Tier 1)"]
        ML["ml_prediction.py\nML models (Tier 2)"]
        EN["enhanced_prediction.py\nStats + Gemini (Tier 3)"]
        ADV["advanced_ml_prediction.py\nNASA + RF + RUL (Tier 4)"]
        PRE["prebuilt_models.py\nLOF + Ridge + SVR (Tier 5)"]
        PJ["prediction_jobs.py\nDispatch: background / Redis"]
    end

    subgraph DB["PostgreSQL (Neon)"]
        SR["sensor_readings"]
        PD["predictions"]
        FL["fault_logs"]
        DV2["devices"]
        CM["commands"]
        US["users"]
    end

    DR --> PJ
    PJ --> PB & ML & ADV
    DR --> FR
    FAST --> DB
    ADV --> AI_CHAIN["Mistral → Gemini → Groq\nFallback Chain"]
```

