# Trajectory Reconstruction

## Objective

Raw aircraft telemetry consists of individual observations. AeroFlux
transforms these observations into ordered trajectory steps and
trajectory segments so that aircraft movement can be analyzed over time.

## Processing Flow

``` text
Aircraft observations
        |
        v
Temporal ordering
        |
        v
Previous-observation comparison
        |
        v
Trajectory-step validation
        |
        v
Movement features
        |
        v
Session / segment construction
        |
        v
4D trajectory
```

## Step Validation

Each observation is evaluated relative to the previous observation.

The final run classified trajectory steps into categories including:

-   `FIRST_OBSERVATION`
-   `GAP_TOO_LARGE`
-   `INVALID_TIME_ORDER`
-   `TOO_CLOSE`
-   `VALID`

The final run contained:

``` text
Trajectory rows:       108,103
Distinct aircraft:         559
Distinct sessions:        1,898
Distinct segments:      24,480
Valid steps:            83,623
Invalid steps:          24,480
```

## Movement Features

Trajectory processing derives movement information including:

-   time delta
-   step distance
-   derived speed
-   speed error ratio
-   acceleration
-   altitude delta
-   vertical acceleration
-   heading change
-   turn rate

These features allow later stages to reason about aircraft movement
rather than isolated coordinates.

## Trajectory Quality

The pipeline assigns quality classifications to trajectory segments.

The final run contained:

``` text
USABLE               98,249
SINGLE_OBSERVATION    9,393
NO_MOVEMENT             461
```

Speed consistency was also classified:

``` text
CONSISTENT            80,880
REVIEW                 1,543
UNKNOWN               25,680
```

The pipeline also verified that invalid segment-efficiency values were
zero in the final quality check.

## 4D Representation

The resulting trajectory represents aircraft movement across:

``` text
latitude
longitude
altitude
time
```

with additional state and movement attributes.

This representation becomes an input to spatial analysis and holding
detection.

## Engineering Considerations

### Irregular telemetry

Aircraft observations are not assumed to arrive at perfectly regular
intervals.

### Invalid temporal relationships

Observations with invalid time ordering are not treated as valid
movement steps.

### Large gaps

Large telemetry gaps are explicitly classified instead of being silently
interpreted as continuous movement.

### Segment boundaries

Trajectory segments provide a controlled unit for downstream analysis.

## Output

The final Gold trajectory dataset contains 108,103 rows for the
processed run.
