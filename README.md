# Senmas Africa

> **Livestock & wildlife digitisation for Botswana and Southern Africa**
> GSM/GPS/RFID tracking • Edge + satellite uplinks • Actuarial risk & pricing models • Farm & insurer APIs

<p align="center">
  <img alt="Senmas Africa logo" src="docs/media/logo-placeholder.png" width="160"/>
</p>

---

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](#)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Made with ❤️ in Botswana](https://img.shields.io/badge/made%20in-Botswana-efefef)](#)

---

## 🔭 Vision

Digitise Botswana’s livestock and wildlife economy with low‑cost sensor devices, resilient connectivity (GSM/LoRaWAN/satellite), and an open analytics platform that delivers:

* **Real‑time tracking & geo‑fencing** for cattle and wildlife.
* **Health & welfare analytics** from movement/rumination/temperature proxies.
* **Actuarial‑grade risk, pricing & capital models** enabling modern insurance (mortality, theft, drought/forage index, liability).
* **Open APIs & dashboards** for farmers, cooperatives, insurers, and government.

## 🧩 Problem & Opportunity

* **Losses:** theft, predation, drought mortality, disease spread, and untraceable movements.
* **Data gaps:** limited traceability across communal & ranch systems.
* **Insurance friction:** sparse data → adverse selection → high prices → low uptake.
* **Opportunity:** cheap sensors + edge ML + satellite uplinks → *continuous risk signals* for pricing, underwriting, and claims.

## 🏗️ Platform Architecture

```mermaid
flowchart LR
  subgraph Field
    D[RFID/GPS Ear Tag\n+ Optional Subcutaneous RFID] -- BLE/LoRa --> G[Edge Collector]
    S[Solar LoRa Gateway] --> G
  end
  G -- GSM/LoRa/Sat --> U[(Uplink Broker)]
  U --> K[(Streaming Bus / Kafka)]
  K --> L[(Lakehouse: Delta/Parquet on S3/GCS)]
  L --> F[Feature Store]
  F --> M[Risk/ML Models]
  M --> A[Actuarial Engine (Pricing/Reserving/SCR)]
  A --> API{{REST/GraphQL API}}
  API --> W[Web & Mobile Dashboards]
  API --> I[Insurer/Partner Integrations]
```

### Components

* **Hardware:** RFID+GPS ear‑tags; collar devices; optional subcutaneous implants.
* **Connectivity:** GSM (2G/4G), LoRaWAN for backhaul, **satellite (BOTSAT‑1)** for remote areas.
* **Edge:** ESP32/STM32 class MCUs; firmware with duty‑cycling & adaptive sampling.
* **Cloud:** object storage (S3/GCS), Kafka, Airflow, Spark, dbt, DuckDB/BigQuery, Postgres.
* **Analytics:** Python/R for GLM, credibility, EVT, Bayesian models; on‑device anomaly detection.
* **Dashboards:** React + Tailwind (farmer) and insurer portals.

## 📦 Repository Layout (proposed)

```
senmas-africa/
├─ firmware/                 # Device code (ESP32/STM32, LoRa, GPS)
├─ edge-gateway/             # Collector + local buffering
├─ infra/                    # IaC (Terraform), networking, secrets
├─ services/
│  ├─ api/                   # FastAPI/Express service
│  ├─ ingestion/             # Kafka connectors, webhooks
│  └─ analytics/             # Notebooks, pipelines, feature store
├─ actuarial/
│  ├─ pricing/               # GLM/GBM pricing, credibility
│  ├─ reserving/             # chain ladder, BF, bootstrap
│  └─ capital/               # SCR proxy models, stress & scenario
├─ web/                      # Farmer & insurer dashboards
├─ data/                     # Sample datasets & schemas
├─ docs/                     # Whitepapers, ADRs, diagrams, media
└─ tools/                    # CLIs, simulators, validators
```

## ⚙️ Getting Started

### Prerequisites

* Python ≥ 3.10, Node ≥ 20, Docker ≥ 24
* An S3/GCS bucket (or MinIO locally)
* Postgres 15+ (TimescaleDB optional)

### Quickstart (local)

```bash
# 1) clone
git clone https://github.com/<org>/senmas-africa.git && cd senmas-africa

# 2) bootstrap dev
make setup  # or: ./tools/dev_bootstrap.sh

# 3) run services
docker compose up -d

# 4) seed sample data
python services/ingestion/seed.py --days 7 --herd 150

# 5) open dashboard
open http://localhost:3000
```

### Configuration

Create `.env` in repo root:

```dotenv
POSTGRES_URL=postgresql://senmas:senmas@localhost:5432/senmas
OBJECT_STORE=s3://senmas-dev
OBJECT_STORE_ENDPOINT=http://localhost:9000   # MinIO
KAFKA_BROKER=localhost:9092
JWT_SECRET=change-me
MAP_TILE_TOKEN=your-map-token
REGION=africa-south1
```

## 🧪 Sample Data & Schemas

**Telemetry** (`data/schemas/telemetry.json`):

```json
{
  "animal_id": "string",
  "ts": "2025-01-01T12:00:00Z",
  "lat": -24.65,
  "lon": 25.92,
  "speed": 1.3,
  "fix_quality": 4,
  "battery_v": 3.75,
  "temp_c": 39.1,
  "geofence_id": "kgotla-north",
  "rssi": -87
}
```

**Herd Register** (`data/schemas/herd_register.csv`):

```
animal_id,sex,breed,dob,owner_id,brand,ear_tag
BW-001,F,Tswana,2022-09-10,owner-42,TM42,ET-0001
```

## 📡 Device → Cloud Ingestion

```python
# services/ingestion/webhook.py (FastAPI)
from fastapi import FastAPI, Request
from datetime import datetime
import json, asyncio

app = FastAPI()

@app.post("/ingest/v1/telemetry")
async def ingest(req: Request):
    payload = await req.json()
    # TODO: validate against JSON schema
    # TODO: push to Kafka topic 'telemetry.raw'
    return {"status": "ok", "received": len(payload)}
```

## 🧠 Analytics & Feature Engineering

```python
# services/analytics/features/motion.py
import pandas as pd

def derive_features(df: pd.DataFrame) -> pd.DataFrame:
    df = df.sort_values(["animal_id","ts"])
    df["dt"] = df.groupby("animal_id")["ts"].diff().dt.total_seconds().fillna(0)
    df["dist_m"] = df["speed"].fillna(0) * df["dt"]
    df["geo_fence_flag"] = df["geofence_id"].notna().astype(int)
    # rolling behaviours
    win = 6*60 // 10  # 10s samples → 1h window
    df["hr_dist_m"] = df.groupby("animal_id")["dist_m"].rolling(win, min_periods=1).sum().reset_index(level=0, drop=True)
    return df
```

## 📊 Actuarial Engine (Pricing, Reserving, Capital)

### Risk Modules

* **Mortality/Theft Frequency:** Poisson/NegBin GLMs with credibility by kraal/brand.
* **Severity:** Gamma/Lognormal; heavy tails via Pareto for catastrophic theft/predation clusters.
* **Drought Index:** NDVI/VCI by grid cell with spatial smoothing; EVT block‑maxima for extremes.
* **Health/Welfare:** anomaly scores from movement & temperature → claim priors.

### Pricing Skeleton (Python)

```python
# actuarial/pricing/base_glm.py
import pandas as pd
import statsmodels.api as sm

X = pd.get_dummies(df[["age_band","breed","region","season"]], drop_first=True)
X = sm.add_constant(X)
y = df["claims_count"]
model = sm.GLM(y, X, family=sm.families.Poisson())
res = model.fit()
quote = float(res.predict(X.iloc[[-1]]) * df.loc[df.index[-1], "sum_insured"])  # simple freq*SI
```

### Reserving & Capital

* **Reserving:** chain ladder, Bornhuetter–Ferguson, bootstrap for IBNR variability.
* **Capital:** SCR proxy using LASSO/XGBoost on simulated losses; drought & price‑shock scenarios.
* **Validation:** out‑of‑sample Gini/HL tests; back‑testing vs. claims; IMV controls.

## 🔌 API (Farmer & Insurer)

```http
POST /ingest/v1/telemetry            # device → cloud
GET  /animals/{id}/trace             # movement history
GET  /risk/quote?animal_id=..        # indicative premium
POST /claims                         # FNOL with evidence links
GET  /insurer/portfolio/metrics      # loss ratios, Gini, drift
```

## 🔐 Security & Privacy

* IoT device keys, mutual TLS, signed firmware updates.
* Role‑based access (RBAC); field‑level encryption for PII.
* Data retention by policy; GDPR/PAIA alignment; animal welfare ethics board.

## 🌍 Compliance & Ethics

* **Animal welfare:** sampling duty cycles to minimise stress & mass; opt‑out by owners.
* **Location privacy:** township‑level aggregation by default; owner consent for sharing.
* **Fair pricing:** model risk governance; bias testing by breed/region/owner type.

## 🚀 Deployment

* **Dev:** Docker Compose + MinIO + LocalStack.
* **Prod:** Terraform into AWS/GCP; MSK/Confluent Kafka; managed Postgres/BigQuery; Grafana/Prometheus; S3 lifecycle rules.

## 🗺️ Roadmap

* [ ] POC: 50‑head pilot, two sites (kraal + ranch); 60‑day data capture.
* [ ] v0.1: Farmer dashboard, basic geofence alerts, NDVI overlay.
* [ ] v0.2: GLM pricing MVP + FNOL flow + parametric drought pilot.
* [ ] v0.3: SCR proxy model, insurer portal, IMV documentation.
* [ ] v1.0: BOTSAT‑1 uplink + national registry integration.

## 🤝 Partners & Stakeholders

* Farmers & cooperatives • Insurers & reinsurers • Ministry of Agriculture • Universities (BIUST, Heriot‑Watt) • Telcos & satellite providers • NGOs

## 📚 Papers & Docs

* `docs/whitepaper.md` — methodology & actuarial governance.
* `docs/adr/` — Architecture Decision Records.
* `docs/api/` — OpenAPI spec.

## 🧰 Development Scripts

```bash
make format     # black + ruff + prettier
make test       # pytest + coverage
make data-lint  # frictionless or great_expectations
```

## 📝 Contributing

1. Fork → feature branch → PR.
2. Add/Update **ADR** for architectural changes.
3. Include tests + docs.
4. Respect the **Code of Conduct**.

## 📄 License

MIT © Senmas Africa

---

### Acknowledgements

This project idea is seeded for Botswana’s livestock & wildlife sectors with the aim of enabling sustainable, data‑driven agriculture and risk transfer.
