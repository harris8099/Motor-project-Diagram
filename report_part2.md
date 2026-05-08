---

## 9. ML Prediction Pipeline

```mermaid
flowchart TD
    INGEST["POST /data — Sensor Payload Received"]
    DISPATCH["prediction_jobs.py\nenqueue_prediction_job()"]
    MODE{"PREDICTION_MODE\nenv variable"}

    BASIC["Tier 1: Rule-Based\nprediction.py"]
    ML["Tier 2: ML Prediction\nml_prediction.py"]
    ADV["Tier 4: Advanced ML\nadvanced_ml_prediction.py"]

    subgraph BASIC_M["Tier 1 — Threshold Rules"]
        B1["Overheating:\navg_temp > maxTempC"]
        B2["Stall Risk:\ncurrent > stallA AND rpm < minRPM"]
        B3["Bearing Fault:\nvibration magnitude > 2.5g"]
        B4["Maintenance Due:\ntotalHours > hoursLimit"]
    end

    subgraph ML_M["Tier 2 — scikit-learn ML"]
        M1["Isolation Forest\nAnomaly Detection\ncontamination=0.1"]
        M2["Linear Regression\nTemp Forecast 6h\n72-step horizon"]
        M3["Linear Regression\nBearing Failure\nZ-score + trend"]
        M4["Polynomial Regression (deg 2)\nEfficiency Degradation"]
        M5["Rule-based Risk Scoring\nStall Risk"]
        M6["Usage Pattern Analysis\nMaintenance Prediction"]
    end

    subgraph ADV_M["Tier 4 — Advanced ML"]
        A1["Isolation Forest +\nOne-Class SVM +\nZ-score Ensemble\nAnomaly Detection"]
        A2["NASA Bearing Health Index\nRMS · Crest Factor\nKurtosis · FFT\nBearing Failure"]
        A3["Linear + Polynomial +\nMoving Average Ensemble\nAdvanced Overheating"]
        A4["Random Forest Classifier\n50 trees · Stall Risk\nFeature Importance"]
        A5["Linear + Exponential\nChange-Point Detection\nEfficiency Degradation"]
        A6["Multi-Indicator Health Index\nTemp + Vib + Current Stress\nRUL Estimation"]
    end

    subgraph STORE["Persist Results"]
        INS["INSERT INTO predictions\n(device_id, type, confidence,\nseverity, details JSONB)"]
        CLEAN["DELETE predictions > 7 days"]
    end

    INGEST --> DISPATCH
    DISPATCH --> MODE
    MODE -->|basic| BASIC --> BASIC_M
    MODE -->|ml| ML --> ML_M
    MODE -->|advanced| ADV --> ADV_M
    BASIC_M & ML_M & ADV_M --> STORE
```

---

## 10. ML Models Reference Table

### Tier 2 — `ml_prediction.py`

| Model | Library | Task | Key Parameters |
|-------|---------|------|----------------|
| **Isolation Forest** | `sklearn.ensemble` | Anomaly detection | `contamination=0.1`, `random_state=42` |
| **StandardScaler** | `sklearn.preprocessing` | Feature normalization | Default |
| **Linear Regression** | `sklearn.linear_model` | Temperature trend forecast | 72-step (6h) horizon |
| **Linear Regression** | `sklearn.linear_model` | Vibration/bearing trend | Z-score + slope analysis |
| **Polynomial Regression** | `numpy.polyfit` | Efficiency curve (degree 2) | Baseline vs. recent comparison |
| **Linear Regression** | `sklearn.linear_model` | Stall risk — efficiency trend | Decreasing efficiency flag |

### Tier 4 — `advanced_ml_prediction.py`

| Model | Library | Task | Key Parameters |
|-------|---------|------|----------------|
| **Isolation Forest** | `sklearn.ensemble` | Ensemble anomaly | `contamination=0.1` |
| **One-Class SVM** | `sklearn.svm` | Ensemble anomaly | `kernel='rbf'`, `nu=0.1` |
| **Random Forest Classifier** | `sklearn.ensemble` | Stall risk classification | `n_estimators=50`, `random_state=42` |
| **Linear Regression** | `sklearn.linear_model` | Overheating (ensemble member) | 72-step horizon |
| **Polynomial Regression** | `numpy.polyfit` | Overheating (ensemble member) | degree=2 |
| **NASA BHI** | `scipy.stats`, `scipy.signal`, `numpy.fft` | Bearing health | RMS, Crest Factor, Kurtosis, FFT |
| **Exponential Smoothing** | Custom (`α=0.3`) | Efficiency time-series | Linear + exponential ensemble |
| **RUL Estimator** | Custom | Remaining Useful Life | Temp + Vib + Current stress indicators |

### Tier 5 — `prebuilt_models.py`

| Model | Library | Task | Key Parameters |
|-------|---------|------|----------------|
| **Local Outlier Factor** | `sklearn.neighbors` | Density anomaly detection | `n_neighbors=20`, `contamination=0.1` |
| **Elliptic Envelope** | `sklearn.covariance` | Gaussian anomaly, Mahalanobis dist | `contamination=0.1` |
| **Ridge Regression** | `sklearn.linear_model` | Temperature prediction | `alpha=1.0` |
| **Lasso Regression** | `sklearn.linear_model` | Power prediction + feature selection | `alpha=0.1` |
| **Gradient Boosting Regressor** | `sklearn.ensemble` | Vibration level prediction | `n_estimators=50` |
| **SVR** | `sklearn.svm` | Efficiency prediction | `kernel='rbf'`, `C=1.0`, `gamma='scale'` |

### AI Commentary Providers

| Provider | Model | Mode | Trigger |
|----------|-------|------|---------|
| **Mistral AI** | `mistral-small` | Commentary / Prediction | Primary (tried first) |
| **Google Gemini** | `gemini-2.0-flash` | Commentary / Prediction | Fallback #2 |
| **Groq** | `llama-3.3-70b-versatile` | Commentary / Prediction | Fallback #3 |

> **Mode `commentary`**: AI provides 2–3 sentence expert commentary on ML alerts.  
> **Mode `prediction`**: AI returns structured JSON — `failure_probability_24h`, `likely_failure_mode`, `maintenance_actions`, `estimated_rul_days`.

---

## 11. Feature Engineering

```mermaid
graph LR
    subgraph RAW["Raw Sensor Inputs (per reading)"]
        R1["power (W)"]
        R2["current (A)"]
        R3["voltage (V)"]
        R4["rpm (int)"]
        R5["temp1 — Motor Temp (°C)"]
        R6["temp2 — Ambient Temp (°C)\n[Open-Meteo if not provided]"]
        R7["accel_x, accel_y, accel_z (g)"]
        R8["energy (kWh)"]
        R9["uptime_seconds"]
        R10["motor_running (bool)"]
    end

    subgraph DERIVED["Derived / Engineered Features"]
        D1["Vibration Magnitude²\n= x² + y² + z²"]
        D2["Motor Efficiency\n= RPM / Power"]
        D3["RMS (NASA)\n= √mean(vib²)"]
        D4["Crest Factor (NASA)\n= peak / RMS"]
        D5["Kurtosis (NASA)\nscipy.stats.kurtosis"]
        D6["FFT Dominant Frequency\nnumpy.fft.fft"]
        D7["Z-score\n= (x − μ) / σ"]
        D8["Efficiency Degradation\n= (baseline − recent) / baseline"]
        D9["Stress Ratio\ntemp + current + vib events / total"]
    end

    subgraph FEAT["10-D Feature Vector (per sample)"]
        FV["[power, current, voltage, rpm,\ntemp1, temp2, vib_mag², energy,\nuptime, motor_status]"]
    end

    RAW --> FEAT
    R7 --> D1 --> FV
    RAW --> DERIVED
```

**Historical Window**: 48–72 hours, up to 200 readings (Tier 2/4), 24 hours for prebuilt models.  
**Minimum samples for ML**: 10 (fallback to rules below this threshold).

---

## 12. Dataset Description

### Source
Real-time sensor data collected from a physical **single-phase induction motor** instrumented with the sensor suite described in Section 4.

### Sensor Specifications

| Sensor | Parameter | Range | Resolution |
|--------|-----------|-------|-----------|
| **PZEM-004T v3.0** | Voltage | 80–260 V AC | 0.1 V |
| | Current | 0–100 A | 0.001 A |
| | Power | 0–23000 W | 0.1 W |
| | Energy | 0–9999.9 kWh | 1 Wh |
| | Frequency | 45–65 Hz | 0.1 Hz |
| | Power Factor | 0.00–1.00 | 0.01 |
| **DS18B20** | Temperature (Motor) | −55 to +125 °C | 0.0625 °C |
| **ADXL345** | Acceleration (X/Y/Z) | ±16 g | 3.9 mg/LSB |
| | Tap Detection | Boolean | — |
| **Hall Effect Sensor** | RPM | 0–∞ | Pulse-counted |
| **Open-Meteo API** | Ambient Temperature | −∞ to +∞ °C | 0.1 °C |

### Database Schema (PostgreSQL — Neon)

```mermaid
erDiagram
    devices {
        TEXT id PK
        TEXT name
        TEXT description
        BOOLEAN is_active
        TIMESTAMPTZ created_at
        TIMESTAMPTZ last_seen_at
        TIMESTAMPTZ last_ai_analysis_at
        TEXT location
        TEXT firmware_version
    }
    sensor_readings {
        BIGSERIAL id PK
        TEXT device_id FK
        TIMESTAMPTZ ts
        BOOLEAN motor_running
        INTEGER rpm
        INTEGER pulse
        BIGINT uptime_seconds
        FLOAT voltage
        FLOAT current
        FLOAT power
        FLOAT power_factor
        FLOAT energy
        FLOAT frequency
        FLOAT temp1
        FLOAT temp2
        FLOAT accel_x
        FLOAT accel_y
        FLOAT accel_z
        BOOLEAN tap_detected
        BOOLEAN fault_overcurrent
        BOOLEAN fault_overtemp
        BOOLEAN fault_stall
        BOOLEAN fault_vibration
        BOOLEAN fault_overvoltage
        BOOLEAN fault_undervoltage
        FLOAT prot_max_current
        FLOAT prot_max_temp
        INTEGER prot_tap_threshold_mg
        INTEGER prot_tap_duration_us
    }
    predictions {
        BIGSERIAL id PK
        TEXT device_id FK
        TIMESTAMPTZ ts
        TEXT prediction_type
        FLOAT confidence
        TEXT severity
        JSONB details
    }
    fault_logs {
        SERIAL id PK
        TEXT device_id FK
        TEXT fault_type
        TEXT severity
        TEXT status
        TEXT message
        TIMESTAMPTZ detected_at
        TIMESTAMPTZ resolved_at
        TEXT root_cause
        TEXT resolution_notes
    }
    commands {
        SERIAL id PK
        TEXT device_id FK
        TEXT command
        JSONB payload
        TEXT status
        TIMESTAMPTZ created_at
        TIMESTAMPTZ acked_at
    }
    devices ||--o{ sensor_readings : "has"
    devices ||--o{ predictions : "has"
    devices ||--o{ fault_logs : "has"
    devices ||--o{ commands : "has"
```

### Data Labeling Strategy
Labels are **self-supervised** — fault ground truth is derived from the hardware protection engine's threshold comparisons on the ESP32 itself. The `fault_*` boolean columns in `sensor_readings` reflect latch states written by `check_motor_protection_internal()`.

### Data Augmentation
`scripts/seed_data.py` generates realistic synthetic readings based on actual motor specifications for initial model warm-up and testing.

---

## 13. Deployment Architecture

```mermaid
graph TB
    subgraph DEV["Developer Machine"]
        CODE["Source Code\n(ESP-IDF + Python + React)"]
        ESPTOOL["esptool.py flash\n→ ESP32 via USB"]
    end

    subgraph ESP_DEPLOY["Physical Hardware"]
        ESPD["ESP32-WROOM-32\nESP-IDF v5.x\nCMake build"]
    end

    subgraph RENDER["Render.com — Backend"]
        WEB["Web Service\nuvicorn app.main:app\n--host 0.0.0.0 --port $PORT"]
        WKR2["Worker Service\npython -m app.worker\n[Redis optional]"]
    end

    subgraph NEON["Neon — Managed PostgreSQL"]
        PG["PostgreSQL 16\nSSL required\nConnection pooling\nPool recycle 300s"]
    end

    subgraph VERCEL["Vercel — Frontend"]
        VB["Vite build\nnpm run build\nSPA rewrites → index.html"]
    end

    CODE --> ESPTOOL --> ESPD
    CODE -->|git push| RENDER
    CODE -->|git push| VERCEL
    RENDER <--> NEON
    ESPD -->|HTTPS POST /data\nx-api-key header| RENDER
    VERCEL -->|REST API\nVITE_API_URL| RENDER
    RENDER -->|Open-Meteo\nambient temp fetch| OMA["Open-Meteo.com\n(no API key)"]
```

### Environment Variables

| Variable | Service | Purpose |
|----------|---------|---------|
| `DATABASE_URL` | Render | Neon PostgreSQL connection string |
| `API_KEY` | Render | Shared key (ESP32 + Frontend) |
| `GEMINI_API_KEY` | Render | Google Gemini (optional) |
| `MISTRAL_API_KEY` | Render | Mistral AI (optional) |
| `GROQ_API_KEY` | Render | Groq (optional) |
| `GEMINI_MODE` | Render | `commentary` / `prediction` / `disabled` |
| `PREDICTION_MODE` | Render | `basic` / `ml` / `advanced` |
| `PREDICTION_DISPATCH_MODE` | Render | `auto` / `background` / `redis` |
| `REDIS_URL` | Render | Optional Redis for worker queue |
| `VITE_API_URL` | Vercel | Render backend URL |
| `VITE_API_KEY` | Vercel | Same shared API key |

---

## 14. Frontend Dashboard Architecture

```mermaid
graph TB
    subgraph APP["React App (src/App.jsx)"]
        AUTH["Auth Guard\nJWT token check\nisAuthenticated()"]
        LOGIN["LoginPage\nJWT login flow"]
        HOME["Home.jsx\nDevice list · Search\nAdd / Bulk Add"]
    end

    subgraph DEVICE["Per-Device Layout (DeviceLayout.jsx)"]
        NAV["Tab Navigation"]
        OV["Overview.jsx\nGauges · Mini charts\nQuick metrics · AI summary"]
        PW["PowerPage.jsx\nVoltage · Current · PF\nEnergy charts"]
        TP["TemperaturePage.jsx\nMotor temp · Ambient temp\nTrend charts"]
        VB["VibrationPage.jsx\nX/Y/Z acceleration\nTap detection"]
        FP["FaultsPage.jsx\nFault lifecycle\nResolve · Root cause"]
        AI["AIPage.jsx\nML predictions\nOn-demand AI analysis"]
        ST["SettingsPage.jsx\nProtection thresholds\nMaintenance config\nCSV export"]
        TM["TerminalPage.jsx\nRaw JSON view\nCommand history"]
    end

    subgraph API["api.js — Frontend API Wrapper"]
        A1["fetchLatestData()"]
        A2["fetchPredictions()"]
        A3["triggerAIAnalysis()"]
        A4["sendCommand()"]
        A5["exportSensorCSV()\nexportFaultLogsCSV()"]
    end

    AUTH --> HOME
    AUTH --> LOGIN
    HOME --> DEVICE
    NAV --> OV & PW & TP & VB & FP & AI & ST & TM
    APP --> API
```

---

## 15. Technology Stack Summary

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Microcontroller** | ESP32-WROOM-32 | — | Edge data collection & control |
| **Firmware Framework** | ESP-IDF | v5.x | FreeRTOS, peripherals, NVS, HTTP |
| **Build System** | CMake | — | Firmware compilation |
| **Power Sensor** | PZEM-004T | v3.0 | Electrical measurements |
| **Temp Sensor** | DS18B20 | — | Motor temperature |
| **Vibration Sensor** | ADXL345 | — | 3-axis acceleration + tap |
| **RPM Sensor** | Hall Effect | — | Magnetic pulse counting |
| **Backend Framework** | FastAPI | latest | REST API, async, OpenAPI |
| **ASGI Server** | Uvicorn | standard | Production HTTP server |
| **ORM / DB Driver** | SQLAlchemy Async + asyncpg | — | PostgreSQL async access |
| **Database** | PostgreSQL (Neon) | 16 | Managed cloud database |
| **ML Library** | scikit-learn | latest | All ML models |
| **Numerical** | NumPy, SciPy | latest | Arrays, FFT, statistics |
| **AI — Primary** | Mistral AI (`mistral-small`) | — | Expert commentary |
| **AI — Secondary** | Google Gemini (`gemini-2.0-flash`) | — | Fallback AI |
| **AI — Tertiary** | Groq (`llama-3.3-70b-versatile`) | — | Fallback AI |
| **Job Queue** | Redis (optional) | — | Out-of-process ML workers |
| **Authentication** | JWT (python-jose + bcrypt) | — | User login + token auth |
| **Frontend Framework** | React | 19 | Dashboard SPA |
| **Build Tool** | Vite | 8 | Fast dev + production build |
| **Charting** | Chart.js / react-chartjs-2 | — | Telemetry visualization |
| **Icons** | Lucide React | — | UI icons |
| **Frontend Hosting** | Vercel | — | SPA CDN deployment |
| **Backend Hosting** | Render.com | — | FastAPI + worker services |
| **Ambient Temp API** | Open-Meteo | — | Free weather API (no key) |
