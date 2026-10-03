# Congestion Forecasting

## Objective

AeroFlux forecasts future aircraft traffic at the H3-cell level using
XGBoost regression.

The model operates on a temporal representation of H3 traffic rather
than directly on raw aircraft observations.

## Pipeline

``` text
Gold H3 traffic
       |
       v
Time-series normalization
       |
       v
Contiguous H3 time blocks
       |
       v
Lag / rolling / flow features
       |
       v
Future targets
       |
       v
Chronological train/test split
       |
       v
XGBoost
       |
       +--------+--------+
       |        |        |
      5 min   15 min   30 min
```

## Temporal Resolution

The forecasting pipeline uses 5-minute traffic windows.

Future targets are generated for:

-   5 minutes
-   15 minutes
-   30 minutes

## Avoiding Artificial Continuity

The pipeline does not assume that every observation for an H3 cell
belongs to one continuous time series.

An internal gap of more than 15 minutes creates a new temporal block.

A minimum history requirement is applied before an H3 block participates
in model training.

This prevents long periods without observations from being interpreted
as continuous history.

## Feature Engineering

### Current state

``` text
unique_aircraft_count
observation_count
mean_altitude_m
mean_velocity_mps
```

### Flow

``` text
inflow_unique_aircraft
outflow_unique_aircraft
net_aircraft_flow
flow_pressure
```

### Lagged features

Examples include:

``` text
traffic_lag_1
traffic_lag_2
traffic_lag_3
traffic_lag_6
traffic_lag_12
```

Additional lagged inflow, outflow, and velocity features are also used.

### Rolling features

The model includes rolling statistics over:

-   15 minutes
-   30 minutes
-   60 minutes

including means, standard deviations, and maximum traffic where
applicable.

### Temporal features

Cyclical time features include:

``` text
hour_sin
hour_cos
dow_sin
dow_cos
```

## Model Evaluation

The model uses a chronological train/test split rather than random
shuffling.

The final run produced:

  ---------------------------------------------------------------------------
  Horizon     Train Rows    Test Rows  XGBoost MAE XGBoost RMSE   Persistence
                                                                  Improvement
  --------- ------------ ------------ ------------ ------------ -------------
  5 min           13,962        4,273       0.2815       0.4451        19.53%

  15 min           9,861        3,028       0.3357       0.4823         2.35%

  30 min           5,107        1,585       0.3269       0.4965         5.46%
  ---------------------------------------------------------------------------

The improvement figures are relative to the persistence baseline used by
the pipeline.

## Dataset Scale

The final forecasting preparation reported:

``` text
Observed global time windows : 103
Eligible H3 cells            : 425
Final grid estimate          : 43,775
Candidate modeling rows      : 43,775
Signal rows                  : 7,016
Zero-only rows               : 36,759
Final pandas modeling rows   : 21,048
Train rows                   : 16,235
Test rows                    : 4,813
Latest forecastable H3 cells : 72
```

These values describe the forecasting preparation and are not equivalent
to aircraft count.

## Forecast Output

The forecast output contains fields such as:

``` text
h3_res_8
prediction_time
forecast_time
horizon_minutes
current_aircraft_count
predicted_aircraft_count
predicted_aircraft_count_rounded
forecast_delta
```

Evaluation records additionally contain:

``` text
actual_aircraft_count
persistence_prediction
absolute_error
```

## Interpretation

The forecasting layer should be interpreted as short-horizon H3 traffic
forecasting.

The final Gold dataset also contains a congestion-signal layer. In the
final run, all 89,643 H3 congestion rows were marked
`INSUFFICIENT_BASELINE`, so the project does not claim that this run
established an operational congestion classification.

## Limitations

-   The system is batch/historical rather than continuously operational.
-   Forecast quality depends on the amount and continuity of historical
    H3 traffic.
-   Sparse cells and zero-only periods reduce the amount of usable
    forecasting signal.
-   Forecasts are analytical estimates, not operational
    air-traffic-control predictions.
