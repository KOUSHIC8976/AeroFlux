# AeroFlux Data Model

## Overview

AeroFlux uses a layered data model that progressively changes the
granularity of the data:

``` text
Aircraft observations
        |
        v
Validated telemetry
        |
        v
Trajectory steps
        |
        v
Trajectory segments / 4D trajectories
        |
        +-------------------+
        |                   |
        v                   v
Airport activity       H3 traffic
                            |
                            v
                       H3 flows
                            |
                            v
                    Forecast features
                            |
                            v
                       XGBoost
```

## Core Identifiers

Important identifiers used throughout the analytical model include:

-   `icao24` --- aircraft identifier
-   `event_time` --- telemetry observation time
-   `flight_session_id` --- reconstructed flight/session grouping
-   `trajectory_segment_id` --- trajectory segment identifier
-   `h3_res_8` --- H3 spatial cell at resolution 8

## Silver Telemetry

The Silver layer enriches observations with movement and quality
attributes.

Representative fields include:

``` text
icao24
event_time
latitude
longitude
baro_altitude_m
velocity_mps
true_track
vertical_rate_mps
previous_event_time
time_delta_seconds
trajectory_step_status
trajectory_step_valid
step_distance_km
derived_speed_mps
speed_error_ratio
speed_consistency_status
acceleration_mps2
altitude_delta_m
vertical_acceleration_mps2
heading_change_deg
turn_rate_deg_per_sec
h3_from_res_8
h3_to_res_8
h3_grid_distance
telemetry_gap
flight_session_id
trajectory_segment_id
segment_duration_minutes
segment_path_length_km
segment_net_displacement_km
segment_trajectory_efficiency
segment_quality
```

## 4D Trajectory

The 4D trajectory representation combines:

``` text
latitude
longitude
altitude
time
```

with additional movement attributes such as velocity, track, vertical
rate, acceleration, heading change, turn rate, H3 cell, session,
segment, and point index.

## H3 Traffic

H3 traffic aggregates observations by spatial cell and time window.

Representative measures include:

-   unique aircraft count
-   observation count
-   inflow
-   outflow
-   net aircraft flow
-   mean altitude
-   mean velocity

## Airport Activity

Airport activity provides a spatially constrained view of aircraft
observations around airport regions.

## Holding Events

Holding events are inferred from trajectory behaviour and include
identifiers, timing, airport association, trajectory efficiency,
heading/turn behaviour, H3 revisitation, observation counts, and holding
confidence/score.

A holding event is an analytical inference, not an official
aviation-certified classification.

## Forecasting Data

The forecasting layer operates on H3 traffic time series.

Representative features include:

-   current traffic
-   observation count
-   inflow/outflow
-   net flow
-   mean altitude
-   mean velocity
-   lagged traffic
-   rolling traffic statistics
-   traffic changes
-   cyclical time features

Targets represent future aircraft counts at 5-, 15-, and 30-minute
horizons.

## Example Gold Output

The final run produced:

  Dataset                 Rows
  ------------------ ---------
  4D trajectory        108,103
  H3 traffic            90,801
  H3 flows              76,796
  Airport activity         144
  Holding events             0
  H3 congestion         89,643

These counts describe the final processed dataset used for the project
evidence.
