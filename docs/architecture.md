# AeroFlux Architecture

## Overview

AeroFlux is a batch airspace-intelligence platform that processes
historical OpenSky aircraft telemetry using Databricks and Apache Spark.

The pipeline progressively transforms raw telemetry into trajectory,
spatial, analytical, and forecasting datasets.

``` text
Historical OpenSky Telemetry
          |
          v
     Bronze Layer
          |
          v
     Silver Layer
          |
     +----+----------------+
     |                     |
     v                     v
4D Trajectories       H3 Traffic
     |                     |
     v                     v
Holding Detection    H3 Congestion Features
     |                     |
     +----------+----------+
                |
                v
           Gold Layer
                |
                v
        XGBoost Forecasting
                |
        +-------+-------+
        |       |       |
       5m      15m     30m
```

## Execution Environment

The implementation is developed and executed in Databricks using Apache
Spark / PySpark.

Delta Lake is used for analytical storage between pipeline stages.

## Layers

### Bronze

The Bronze layer ingests and stores historical OpenSky telemetry.

Its purpose is to establish a durable raw representation before
analytical transformations are applied.

### Silver

The Silver layer cleans, validates, orders, and enriches telemetry.

Examples of derived information include:

-   trajectory step validity
-   time gaps
-   movement distance
-   derived speed
-   speed consistency
-   acceleration
-   altitude changes
-   heading changes
-   H3 mappings
-   flight sessions
-   trajectory segments
-   segment quality

### Gold

The Gold layer contains analytical products:

-   4D trajectories
-   H3 traffic
-   H3 flows
-   airport activity
-   holding-event analysis
-   H3 congestion/forecasting datasets

## Forecasting Layer

The forecasting pipeline consumes Gold H3 traffic data and constructs
temporal features before training XGBoost models.

Forecast horizons are:

-   5 minutes
-   15 minutes
-   30 minutes

The forecasting pipeline uses chronological train/test separation and
evaluates predictions against a persistence baseline.

## Design Principles

### Layered transformations

Each layer creates a more useful representation of the data without
mixing raw ingestion, trajectory logic, and forecasting concerns.

### Spatial-temporal representation

Aircraft movement is represented through both trajectory/time
information and H3 spatial cells.

### Defensive time-series construction

H3 observations are separated into contiguous time blocks so large gaps
are not incorrectly treated as continuous history.

### Evidence-driven analytics

Derived events such as holding behaviour are only produced when the
defined trajectory criteria are satisfied.

## Scope

AeroFlux is a historical/batch analytical platform. It is not presented
as a continuously operating aviation control system.
