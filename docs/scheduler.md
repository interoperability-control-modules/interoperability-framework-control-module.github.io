# Scheduler Agent

[Code & Installation Instructions](https://github.com/interoperability-control-modules/scheduler){ .md-button }

The scheduler agent plays a vital role within the VOLTTRON agents' framework, facilitating efficient energy management
and ensuring seamless integration with other service agents. The agent is primarily responsible for scheduling energy
storage and grid operations by leveraging forecasted data on demand, generation, and pricing. It is supported by a
modular architecture allowing for incorporating various algorithmic approaches to optimize energy scheduling
efficiently, including optimization-based and machine-learning-based schedulers.

The scheduler agent operates by interacting with multiple system components, such as Photovoltaic (PV) systems and
Energy Storage Systems (ESS), through the [Interoperability Service](interoperability-service.md), which translates
native device data models (SunSpec Modbus, IEEE 1815.2, IEEE 2030.5, IEC 61850) into a common format. It consumes
price and CO₂ signals from the [Grid Signals](grid-signals.md) agent and load forecasts from the
[Forecaster Agent](forecaster-agent.md), and its hourly dispatch is followed in real time by the
[Real-Time Control Agent](rt-control.md) through the Charge/Discharge Storage or rule-based modes.

## Operating methods

The agent behaves differently depending on the configured `method`:

* **`control`**: runs the optimizer over the forecasted price, load, and current state of charge to compute an hourly
  dispatch, schedules actuation of the ESS at each hour of the horizon, and publishes a schedule record.
* **`schedule`**: uses precomputed `bess_setpoints` or `tess_setpoints`, rotated to start at the current hour,
  schedules actuation at each hour, and publishes the set points.
* **`direct`**: immediately actuates the ESS with the fixed `tess_direct_signal` value.

The optimizer minimizes electricity cost over the scheduling horizon using the forecast price signal and demand
charge structure, subject to the battery power limits, state-of-charge bounds, and charging and discharging
efficiencies. Battery (BESS), thermal (TESS), and hybrid storage systems are supported.

## Published schedule

For the `control` and `schedule` methods, the agent publishes a schedule message on
`record/<campus>/<building>/<device>/schedule`. The message maps each period start time to a dictionary describing
that period:

| Period Key            | Description                                                                              |
|-----------------------|------------------------------------------------------------------------------------------|
| `<ess>_setpoint`      | The scheduled active-power set point in kW (e.g. `bess_setpoint`).                       |
| `duration_in_seconds` | How long the set point applies, in seconds (default `3600`).                             |

The scheduler actuates the ESS directly through the configured actuator agents; the published message serves as a
record of the computed schedule for historians and other subscribers, such as the Real-Time Control Agent.

## Configuration

| Parameter                       | Description                                                                                                   |
|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| `campus`, `building`, `device`  | Identifiers used to build the state-of-charge read path and the schedule publish topic.                       |
| `energy_storage_system`         | The system being scheduled: `bess`, `tess`, or `hybrid`.                                                      |
| `method`                        | Operating mode of the agent: `control`, `schedule`, or `direct`.                                              |
| `run_schedule`                  | Cron expression for when the optimization runs (default `0 0 * * *`, daily at midnight).                      |
| `window_length`                 | Number of hourly steps in the scheduling horizon (default `24`).                                              |
| `soc_point_name`                | Point name used to read state of charge (default `BAT_SOC`).                                                  |
| `bess_actuator_vip`, `tess_actuator_vip` | VIP identities of the agents used to actuate the BESS and TESS.                                      |
| `bess_setpoints`, `tess_setpoints` | Precomputed hourly set points used when `method` is `schedule`.                                            |
| `tess_direct_signal`            | The fixed set point actuated when `method` is `direct`.                                                       |
| `season`                        | Season used by the optimizer (e.g. `Summer`).                                                                 |
| `data_source`                   | Historian identity used for retrieving data (default `postgres.cetc`).                                        |
| `weather_vip`                   | VIP identity of the weather agent (default `platform.weather`).                                               |
| `forecast_config`               | Forecast inputs: `forecast_data_source` (`info_agent` to subscribe to live topics), `price_topic`/`price_point`, `load_forecast_topic`/`load_forecast_point`, or static `predicted_price`/`predicted_load` arrays. |
| `bess_optimizer_config`         | Battery parameters: rated kW and kWh, initial, minimum, maximum, and reserve SOC, charging and discharging efficiencies, demand charge. |
| `demand_rate_config`            | Tariff structure: peak and partial-peak periods, demand charges, and energy prices per period.                |

### Example

```json
{
  "campus": "PNNL",
  "building": "SEB",
  "device": "BESS",
  "energy_storage_system": "bess",
  "method": "control",
  "bess_actuator_vip": "bess.control",
  "run_schedule": "0 0 * * *",
  "window_length": 24,
  "season": "Summer",
  "forecast_config": {
    "forecast_data_source": "info_agent",
    "price_topic": "devices/PNNL/grid_information/price/all",
    "price_point": "tou",
    "load_forecast_topic": "devices/PNNL/SEB/forecast/all",
    "load_forecast_point": "load"
  },
  "bess_optimizer_config": {
    "bess_rated_kw": 100,
    "initial_soc": 50,
    "min_soc": 10,
    "max_soc": 90,
    "demand_charge": 26.06,
    "bess_parameter": {
      "bess_chg_max": 100,
      "bess_dis_max": 100,
      "bess_reserve_soc": 20,
      "bess_rated_kWh": 200,
      "charging_efficiency": 0.95,
      "discharging_efficiency": 0.975
    }
  }
}
```

## Requirements

* Python >= 3.10
* VOLTTRON >= 10.0

## Installation

Before installing, VOLTTRON should be installed and running with its virtual environment active.

```shell
git clone https://github.com/interoperability-control-modules/scheduler
vctl install ./scheduler --vip-identity agent.scheduler --tag scheduler --start
vctl config store agent.scheduler config path/to/config.json
```
