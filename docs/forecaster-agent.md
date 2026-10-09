# Forecaster Agent

[Code & Installation Instructions](https://github.com/interoperability-control-modules/load-forecaster){ .md-button }

The Forecaster Agent (Load Forecaster) is a VOLTTRON agent that produces short-term forecasts of building electrical
and cooling load for use by the [Scheduler](scheduler.md) and other Interoperability Framework and Control Module applications. It utilizes historical
data from the VOLTTRON historian, a weather forecast, and a thermal model to predict future energy load and outdoor
temperature conditions, helping anticipate grid needs and optimize the scheduling and operation of energy storage
systems. Forecasts are published on the VOLTTRON message bus, where they are consumed by the Scheduler and recorded
for visualization alongside the [Grid Signals](grid-signals.md).

## How it works

1. **Historical data.** The agent queries the historian (by default a PostgreSQL historian, optionally on an external
   VOLTTRON platform) for the whole-building power and the BESS power over a configurable collection window, and
   subtracts the storage contribution to obtain the uncontrollable building load.
2. **Weather forecast.** A 24-hour outdoor air temperature forecast is obtained from the VOLTTRON weather agent for
   the configured location.
3. **Load prediction.** A building load model predicts the hourly load profile for the coming forecast window
   (24 hours by default) from the historical profile for the current weekday, the season, and the forecast
   temperature. A thermal model of the building and its air handling units estimates the cooling load.
4. **Publication.** One message per forecast hour is published, with a `Date` header giving the hour the forecast
   applies to.

## Published topic

| Topic                                     | Point  | Description                                        |
|-------------------------------------------|--------|----------------------------------------------------|
| `devices/<campus>/<building>/forecast/all` | `load` | Forecasted building load for the hour, in kW.      |

The Scheduler subscribes to this topic through its `load_forecast_topic` and `load_forecast_point` settings.

## Configuration

| Parameter                  | Description                                                                                         |
|----------------------------|-----------------------------------------------------------------------------------------------------|
| `campus`, `building`, `device` | Identifiers used to build the publish topic.                                                    |
| `run_schedule`             | Cron expression for when the forecast is produced (default daily at midnight).                      |
| `window_length`            | Number of hourly steps in the forecast horizon (default `24`).                                      |
| `data_source`              | VIP identity of the historian used for historical data (default `postgres.cetc`).                   |
| `external_platform`        | Name of the external VOLTTRON platform hosting the historian, for cross-platform RPC.               |
| `load_topic`               | Historian topic of the whole-building power measurement.                                            |
| `duration_data_collection` | Number of days of historical data to retrieve.                                                      |
| `start_day_data_collection` | Number of days before today at which the collection window ends (default `1`).                     |
| `location`                 | Weather forecast location, as a list of `{"wfo", "x", "y"}` National Weather Service grid points.   |
| `weather_vip`              | VIP identity of the weather agent (default `platform.weather`).                                     |
| `season`                   | Season used to select the load profile (e.g. `Summer`).                                             |
| `load_file`                | CSV power profile used when historian data is unavailable.                                          |

### Example

```json
{
  "campus": "PNNL",
  "building": "SEB",
  "device": "BESS",
  "run_schedule": "0 0 * * *",
  "data_source": "postgres.cetc",
  "external_platform": "vc",
  "load_topic": "PNNL/SEB/ELECTRIC_METER/WholeBuildingPower",
  "duration_data_collection": 20,
  "location": [{"wfo": "PDT", "x": 119, "y": 131}],
  "season": "Summer"
}
```

## Requirements

* Python >= 3.10
* VOLTTRON >= 10.0
* `pandas`, `numpy`, `matplotlib`, `pyomo`
* A VOLTTRON historian holding the building and BESS power history, and a VOLTTRON weather agent

## Installation

Before installing, VOLTTRON should be installed and running with its virtual environment active.

```shell
git clone https://github.com/interoperability-control-modules/load-forecaster
vctl install ./load-forecaster --tag load-forecaster --start
```
