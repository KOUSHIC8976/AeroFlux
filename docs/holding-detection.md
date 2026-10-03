# Holding Detection

## Objective

AeroFlux contains a trajectory-based detector for identifying aircraft
movement that exhibits characteristics associated with holding behaviour
near airports.

The detector is intentionally described as an **inferred analytical
event**, not an official aviation-certified holding classification.

## Detection Flow

``` text
4D trajectory
      |
      v
Airport proximity
      |
      v
Trajectory segment analysis
      |
      +-----------------------------+
      |             |               |
      v             v               v
Duration      Heading/turns    H3 revisitation
      |             |               |
      +-------------+---------------+
                    |
                    v
             Holding score
                    |
                    v
          Holding candidate
                    |
                    v
          Gold holding event
```

## Candidate Characteristics

The holding detector evaluates characteristics including:

-   minimum duration
-   observation count
-   airport-region proximity
-   trajectory efficiency
-   repeated heading changes
-   turn behaviour
-   H3 cell revisitation
-   vertical movement behaviour
-   airborne evidence

## Airport Region

Holding analysis is constrained to aircraft observations within the
configured airport region.

The detector uses airport proximity and previous-airport-distance
information when determining whether local trajectory steps are valid
for the analysis.

## Gold Holding Event

The final holding-event output contains fields including:

``` text
holding_event_id
icao24
airport
trajectory_segment_id
representative_h3_res_8
holding_start
holding_end
duration_minutes
minimum_airport_distance_km
path_length_km
net_displacement_km
trajectory_efficiency
mean_altitude_m
altitude_stddev_m
mean_abs_vertical_rate_mps
large_heading_change_count
large_turn_count
unique_h3_cells
h3_revisit_ratio
observation_count
valid_local_steps
airborne_status
holding_score
holding_confidence
detection_version
```

## Final Run

The final dataset produced:

``` text
Holding events detected: 0
```

This means no trajectory segment in the processed sample satisfied the
complete holding-candidate criteria.

The detector therefore did not manufacture an event simply to populate
the output.

## Interpretation

A zero-event result is a valid analytical outcome.

The holding detector is dependent on the characteristics and temporal
coverage of the input sample. A larger or different historical sample
may produce qualifying events.

## Relationship to Congestion

When holding events exist, holding activity can be aggregated by H3 cell
and time window.

The resulting measures can contribute to H3 congestion analysis through:

-   holding aircraft count
-   holding event count

This connects trajectory-level behaviour with spatial traffic
intelligence.
