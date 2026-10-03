# AeroFlux

### Batch Airspace Intelligence & Short-Horizon Congestion Forecasting

AeroFlux is a **batch airspace-intelligence platform** built on historical OpenSky aircraft telemetry.

It transforms raw aircraft state observations into reconstructed **4D trajectories**, **H3-based spatial traffic intelligence**, **inferred holding events**, and **short-horizon congestion forecasts** using XGBoost.

The project focuses on the engineering problem of turning noisy, irregular aircraft telemetry into structured spatial-temporal intelligence.

---

## What AeroFlux Does

AeroFlux answers four progressively higher-level questions:

1. **Where did each aircraft move?**
   - Reconstruct aircraft trajectories from timestamped state observations.
   - Derive movement and trajectory-quality characteristics.

2. **Where is air traffic concentrated?**
   - Map observations into H3 spatial cells.
   - Aggregate aircraft activity into time buckets.
   - Measure traffic, inflow, outflow, altitude, and velocity.

3. **Where does aircraft behavior indicate holding?**
   - Detect candidate holding behavior from trajectory geometry, temporal persistence, airport proximity, heading changes, H3 revisitation, and airborne evidence.
   - Store the result as an inferred analytical event.

4. **Where is traffic likely to increase?**
   - Build regular H3 time series.
   - Generate lag, rolling, flow, trend, and cyclic time features.
   - Forecast future aircraft counts at **5, 15, and 30 minute horizons** with XGBoost.

---

## Architecture

```text
                    Historical OpenSky Telemetry
                              │
                              ▼
                       Batch Ingestion
                              │
                              ▼
                    ┌─────────────────────┐
                    │   Spark / Delta     │
                    │   Processing Layer  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │      Silver         │
                    │ Quality / Normalize │
                    │ Dedup / Trajectory  │
                    └──────────┬──────────┘
                               │
                 ┌─────────────┼─────────────┐
                 │             │             │
                 ▼             ▼             ▼
          4D Trajectories   H3 Traffic   Airport Context
                 │             │             │
                 │             ▼             │
                 │      Traffic Features     │
                 │             │             │
                 ▼             ▼             │
          Holding Detection   XGBoost ◄──────┘
                 │             │
                 ▼             ▼
        Gold Holding Events   Congestion
                              Forecasts
                 │             │
                 └──────┬──────┘
                        ▼
                 Airspace Intelligence
```

### Data flow

```text
OpenSky historical states
        │
        ▼
     Bronze
        │
        │ quality + normalization
        ▼
     Silver
        │
        ├──────────────► 4D trajectory reconstruction
        │                         │
        │                         ▼
        │                  Holding detection
        │
        └──────────────► H3 spatial traffic
                                  │
                                  ▼
                         Temporal feature engineering
                                  │
                                  ▼
                           XGBoost forecasting
                                  │
                                  ▼
                         5 / 15 / 30 min forecasts
```

---

## Core Engineering

### 1. 4D Trajectory Reconstruction

Aircraft telemetry is not treated as an isolated collection of rows.

For each aircraft, observations are ordered in time and used to reconstruct a trajectory containing:

- Position
- Timestamp
- Altitude
- Velocity
- Heading / track-derived movement
- Temporal movement characteristics
- Trajectory segment information
- H3 spatial representation
- Airport proximity information

The resulting trajectory representation provides the foundation for downstream spatial and behavioral analytics.

Conceptually:

```text
Aircraft
   │
   ├── t0 → position / altitude / velocity / heading
   ├── t1 → position / altitude / velocity / heading
   ├── t2 → position / altitude / velocity / heading
   └── ...
             │
             ▼
       4D trajectory
       (x, y, z, t)
```

---

### 2. H3 Spatial Traffic Intelligence

AeroFlux uses **H3 resolution 8** as the spatial indexing layer.

Aircraft observations are mapped to H3 cells and aggregated into temporal windows.

The Gold traffic layer contains metrics including:

- `unique_aircraft_count`
- `observation_count`
- `inflow_unique_aircraft`
- `outflow_unique_aircraft`
- `net_aircraft_flow`
- `mean_altitude_m`
- `mean_velocity_mps`

This converts individual aircraft observations into a spatial-temporal representation of air traffic.

```text
Aircraft observations
        │
        ▼
     H3 cells
        │
        ▼
   5-minute windows
        │
        ▼
Spatial traffic state
```

---

### 3. Holding Event Detection

Holding is treated as an **inferred trajectory behavior**, not as an official aviation-certified holding classification.

The detector evaluates trajectory evidence such as:

- Minimum duration
- Observation count
- Airport-region proximity
- Trajectory efficiency
- Heading changes
- Turn behavior
- H3 cell revisitation
- Altitude behavior
- Vertical-rate behavior
- Airborne status

A candidate event is assigned:

- `holding_event_id`
- `holding_score`
- `holding_confidence`
- `detection_version`

The current detector is explicitly versioned as:

```text
holding-v2
```

This makes the analytical output reproducible and allows future detector versions to coexist conceptually without silently changing historical semantics.

Holding events also feed the spatial congestion layer, allowing holding activity to contribute to H3-level traffic intelligence.

---

## Congestion Forecasting

AeroFlux forecasts future aircraft occupancy for each H3 cell.

### Horizons

| Horizon | Prediction |
|---|---|
| 5 min | Near-term aircraft count |
| 15 min | Short-horizon aircraft count |
| 30 min | Extended short-horizon aircraft count |

### Model

**XGBoost regression**

The model uses a chronological train/test split rather than randomly shuffling temporal observations.

### Feature groups

#### Current traffic
- Current aircraft count
- Observation count

#### Flow
- Inflow
- Outflow
- Net flow
- Flow pressure

#### State
- Mean altitude
- Mean velocity

#### Lag features
- 5 min
- 10 min
- 15 min
- 30 min
- 60 min

#### Rolling features
- 15-minute mean/std
- 30-minute mean/std/max
- 60-minute mean/std

#### Trend features
- 5-minute change
- 15-minute change
- 30-minute change

#### Temporal features
- Hour-of-day cyclic encoding
- Day-of-week cyclic encoding

---

## Temporal Integrity

A major design constraint is avoiding false continuity in irregular telemetry.

AeroFlux therefore does **not** blindly treat all observations for an H3 cell as one continuous time series.

For H3 forecasting:

- Traffic is normalized into temporal windows.
- Large gaps create separate time-series blocks.
- Internal gaps within the configured tolerance can be represented as zero-traffic windows.
- Lag features are accepted only when their actual timestamp distance matches the intended interval.
- Forecast targets are generated from future observations at the required horizon.

This prevents a missing or disconnected period from silently becoming a misleading lag relationship.

---

## Model Evaluation

The forecasting layer evaluates each horizon against a **persistence baseline**.

Reported metrics include:

- MAE
- RMSE
- Persistence MAE
- Persistence RMSE
- Improvement over persistence
- Training/test row counts
- Feature count
- Train/test temporal ranges

The forecasting layer also records feature importance for each horizon.

No performance numbers are hard-coded into this README; the values produced by the current dataset run should be taken directly from the generated metrics output.

---

## Gold Outputs

The project produces analytical Gold datasets for the major stages.

### Trajectory intelligence

```text
gold_4d_trajectory
```

Contains reconstructed aircraft trajectory information.

### Holding intelligence

```text
gold_holding_events
```

Contains inferred holding events and their trajectory-derived characteristics.

### Spatial traffic

```text
gold_h3_traffic
```

Contains H3/time-window traffic aggregates used by the forecasting layer.

### Congestion forecasting

Depending on the final forecasting notebook configuration, the forecasting layer produces:

```text
xgb_congestion_forecast
xgb_congestion_latest
xgb_congestion_metrics
xgb_congestion_feature_importance
```

and trained XGBoost model artifacts.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Source data | OpenSky historical aircraft telemetry |
| Processing | Apache Spark / PySpark |
| Storage | Delta Lake |
| Spatial indexing | H3 |
| Data processing | Python / Pandas / NumPy |
| Machine learning | XGBoost |
| Evaluation | scikit-learn |
| Execution environment | Databricks |

---

## Why These Technologies

### Apache Spark

Used for distributed transformation of aircraft telemetry, temporal ordering, aggregation, window operations, and analytical feature engineering.

### Delta Lake

Provides the table/storage abstraction used by the pipeline's analytical layers.

### H3

Provides a hierarchical spatial index that turns latitude/longitude observations into a consistent spatial representation suitable for aggregation and forecasting.

### XGBoost

Provides a strong tabular regression model for short-horizon congestion forecasting while keeping the feature engineering explicit and inspectable.

---

## Project Structure

A representative repository structure is:

```text
aeroflux/
│
├── README.md
├── requirements.txt
│
├── notebooks/
│   ├── bronze/
│   ├── silver/
│   ├── gold/
│   ├── trajectory/
│   ├── holding/
│   └── forecasting/
│
├── docs/
│   └── architecture/
│
└── ...
```

The exact notebook/file organization may differ depending on the Databricks workspace layout.

---

## Running the Project

AeroFlux is designed as a **batch/historical analytics project** and is intended to run in a Databricks + Spark environment.

### 1. Install Python dependencies

```bash
pip install -r requirements.txt
```

Spark and Delta Lake are expected to be provided by the Databricks runtime.

### 2. Prepare historical OpenSky telemetry

Place the historical telemetry in the input location expected by the ingestion layer.

### 3. Execute the pipeline

Run the processing stages in dependency order:

```text
Raw telemetry
     │
     ▼
Bronze
     │
     ▼
Silver / quality + normalization
     │
     ├──────────────► 4D trajectory
     │                      │
     │                      ▼
     │                Holding events
     │
     └──────────────► H3 traffic
                            │
                            ▼
                     XGBoost forecasting
```

### 4. Inspect Gold outputs

The final analytical outputs can then be queried from the Gold layer.

---

## Important Scope

AeroFlux is intentionally scoped as a **batch historical airspace-intelligence platform**.

It is **not presented as a continuously running operational air-traffic-control system**.

The current implementation does not claim:

- 24/7 live telemetry processing
- Operational ATC decision authority
- Official aviation holding certification
- Real-time operational predictions
- Fuel or CO₂ estimation

The current project scope is:

> **Historical aircraft telemetry → trajectory reconstruction → spatial traffic intelligence → inferred holding detection → short-horizon congestion forecasting.**

This keeps the engineering claims aligned with what the implemented pipeline actually produces.

---

## Design Decisions

### Historical batch processing instead of simulated real-time

The project uses historical telemetry and therefore prioritizes:

- Reproducibility
- Temporal correctness
- Replayability
- Analytical depth
- Deterministic transformations

Rather than presenting a batch workload as a live operational system.

### H3 as the spatial abstraction

Aircraft-level telemetry is difficult to model directly at network scale.

H3 provides a common spatial key that allows traffic to be transformed into time-indexed spatial states.

### Chronological model evaluation

Random train/test splitting would allow future temporal information to leak into the training population.

A chronological split is therefore used for forecasting evaluation.

### Persistence baseline

A machine-learning model should be compared against a simple temporal baseline.

A persistence forecast provides that reference:

```text
future traffic ≈ current traffic
```

The XGBoost metrics are therefore interpreted relative to a simple non-ML forecasting strategy.

### Versioned holding detection

Holding detection is an inference problem based on telemetry-derived behavior.

The detector therefore carries an explicit detection version:

```text
holding-v2
```

---

## Analytical Outputs

The system ultimately produces three complementary views of the airspace:

```text
                    AIRSPACE INTELLIGENCE
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
     TRAJECTORIES       BEHAVIOR        FORECASTING
          │                │                │
          ▼                ▼                ▼
       4D path          Holding          Future
      reconstruction     events          traffic
                           │                │
                           └───────┬────────┘
                                   ▼
                          Spatial-temporal
                             intelligence
```

This makes the project more than a conventional ETL pipeline: the engineering pipeline produces structured analytical representations that can support downstream airspace analysis.

---

## Limitations

The system is an analytical/research implementation and should be interpreted accordingly.

### Data limitations

OpenSky telemetry represents observed aircraft state information and may contain:

- Missing observations
- Irregular sampling
- Data gaps
- Position uncertainty
- Incomplete aircraft context

### Holding detection limitations

Holding events are inferred from observed trajectory behavior.

They should not be interpreted as authoritative operational ATC classifications.

### Forecasting limitations

The XGBoost model forecasts aircraft counts from historical spatial-temporal features.

It does not model every external factor that can influence future air traffic, such as:

- Weather
- Filed flight plans
- Airport scheduling
- Airspace restrictions
- Controller interventions
- Airline operational decisions

### Operational limitation

The system is not an operational aviation safety system and should not be used for live ATC decisions.

---

## Reproducibility

Key pipeline behavior is explicitly parameterized, including:

- Forecast horizons
- Time-step size
- Maximum internal temporal gap
- Minimum valid H3 block length
- Chronological train/test ratio
- Random seed
- Holding detector version

This makes the analytical pipeline easier to rerun against another historical dataset and easier to reason about when outputs change.

---

## Future Extensions

The architecture can support future research modules without changing the core analytical foundation.

Potential extensions include:

- Real-time telemetry ingestion
- Streaming state management
- Advanced trajectory prediction
- Temporal graph models
- Weather-aware forecasting
- Capacity-aware congestion modeling
- Counterfactual routing and delay simulation
- Network optimization
- Operational visualization

These are **future extensions**, not claims about the current implementation.

---

## Summary

AeroFlux turns aircraft telemetry into a progressively richer representation of airspace:

```text
Raw telemetry
      │
      ▼
Cleaned observations
      │
      ▼
4D aircraft trajectories
      │
      ├───────────────► Inferred holding events
      │
      ▼
H3 spatial traffic
      │
      ▼
Temporal feature engineering
      │
      ▼
XGBoost
      │
      ├── 5 min forecast
      ├── 15 min forecast
      └── 30 min forecast
```

**The core engineering problem is simple to state:**

> **Convert irregular aircraft telemetry into reliable spatial-temporal intelligence that can describe current traffic behavior and forecast near-term airspace demand.**

AeroFlux implements that pipeline using **Spark, Delta Lake, H3, trajectory reconstruction, behavioral detection, and XGBoost**.
