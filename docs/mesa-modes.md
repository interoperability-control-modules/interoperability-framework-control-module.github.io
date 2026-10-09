# MESA Modes

[Code & Installation Instructions](https://github.com/interoperability-control-modules/realtime-control-agent){ .md-button }

The MESA-ESS specification (Modular Energy Storage Architecture), built on IEEE 1815.2 (DNP3) and the
IEC 61850-7-420 DER information model, defines a set of standard control modes for energy storage systems.
The [Real-Time Control Agent](rt-control.md) implements these modes natively in Python as subclasses of a common
`MesaMode` base class, so that a storage system can provide standards-aligned grid services regardless of whether the
device itself supports the mode. Modes are grouped into active power, reactive power, and emergency families, as
shown in [](#mesa-active), [](#mesa-reactive), and [](#mesa-emergency).

Several modes may be active at once. Limiting modes such as Active Power Limit constrain the commands produced by the
other active modes, and the reactive power modes operate independently of the active power modes.

## Active power modes

Table: MESA active power modes implemented in the Real-Time Control Agent {#mesa-active}

| Control Mode             | Description                                                                                                                                                                                                                                                                                   |
|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Active Power Limit       | Disallows power commands outside of specified charge and discharge percentages of the rated power. This is used along with other control modes to constrain their output commands.                                                                                                             |
| Active Power Response    | Responds to a measured power signal once it exceeds an activation threshold, at a configured output ratio. This mode may be configured to provide Generation Following, Load Following, or Peak Limiting, depending on the Use Case with which it is paired.                                     |
| Active Power Smoothing   | Actuates the storage using a moving average filter applied to the measured output of a variable resource (e.g., the production meter on a photovoltaic array), reducing ramp rates seen by the grid.                                                                                           |
| AGC                      | Follows an Automatic Generation Control command signal from the system operator within configured state-of-charge limits.                                                                                                                                                                       |
| Charge/Discharge Storage | The battery is actuated using an ingested schedule (for example from the [Scheduler](scheduler.md)) or a pre-configured active power target, within configured reserve limits.                                                                                                                  |
| Frequency-Watt           | Adjusts power in response to a measured frequency signal to support the nominal grid frequency. **Vertex mode** uses a configured piecewise linear curve with optional hysteresis curves; **Gradient mode** uses continuous, proportional charge, discharge, and return gradients with start and stop frequencies and delays. |
| Volt-Watt                | Reduces active power output (or increases charging) as the measured voltage rises above a configured curve, to mitigate over-voltage at the point of connection.                                                                                                                                 |

## Reactive power modes

Table: MESA reactive power modes implemented in the Real-Time Control Agent {#mesa-reactive}

| Control Mode                    | Description                                                                                                                                                   |
|---------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Constant VAR                    | Holds reactive power output at a fixed setting.                                                                                                                |
| Fixed Power Factor              | Holds the power factor of the inverter at a fixed setting, adjusting reactive power as active power varies.                                                    |
| Power Factor Correction         | Adjusts reactive power to maintain a target power factor at the reference meter, compensating for the reactive demand of other loads.                          |
| Volt-VAR                        | Adjusts reactive power as a function of the measured voltage according to a configured curve, to support voltage regulation.                                    |
| Watt-VAR                        | Adjusts reactive power as a function of the active power output according to a configured curve.                                                               |
| Dynamic Reactive Current Support | Injects or absorbs reactive current in response to rapid deviations of the voltage from its moving average, supporting the grid during voltage transients.     |

## Emergency modes

Table: MESA emergency modes implemented in the Real-Time Control Agent {#mesa-emergency}

| Control Mode            | Description                                                                                                                                              |
|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------|
| Frequency Ride-Through  | Keeps the storage system connected and operating through frequency excursions according to configured ride-through and trip regions.                     |
| Voltage Ride-Through    | Keeps the storage system connected and operating through voltage excursions according to configured ride-through and trip regions.                       |

## Mode parameters

Each mode is configured as an entry of the `modes` list in the Real-Time Control Agent configuration, with
`class_name` set to the mode name and the parameters listed in [](#mesa-params).

Table: Configuration parameters of the MESA active power modes {#mesa-params}

| Control Mode             | Parameters                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Active Power Limit       | &#x2022;&nbsp;maximum_charge_percentage: float<br/>&#x2022;&nbsp;maximum_discharge_percentage: float                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Active Power Response    | &#x2022;&nbsp;activation_threshold: float<br/>&#x2022;&nbsp;output_ratio: float<br/>&#x2022;&nbsp;ramp_params: dict                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Active Power Smoothing   | &#x2022;&nbsp;smoothing_gradient: float<br/>&#x2022;&nbsp;lower_smoothing_limit: float<br/>&#x2022;&nbsp;upper_smoothing_limit: float<br/>&#x2022;&nbsp;smoothing_filter_time: float or timedelta                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| AGC                      | &#x2022;&nbsp;minimum_usable_soc: float<br/>&#x2022;&nbsp;maximum_usable_soc: float                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Charge/Discharge Storage | &#x2022;&nbsp;minimum_reserve_percent: float = 10.0<br/>&#x2022;&nbsp;maximum_reserve_percent: float = 90.0<br/>&#x2022;&nbsp;active_power_target: float or None = None                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Frequency-Watt           | &#x2022;&nbsp;use_curves: bool<br/>&#x2022;&nbsp;frequency_watt_curve, low_hysteresis_curve, high_hysteresis_curve: list of (frequency, power) vertices<br/>&#x2022;&nbsp;start_delay, stop_delay: float or timedelta<br/>&#x2022;&nbsp;minimum_soc, maximum_soc: float<br/>&#x2022;&nbsp;use_hysteresis, use_snapshot_power: bool<br/>&#x2022;&nbsp;high_starting_frequency, low_starting_frequency, high_stopping_frequency, low_stopping_frequency: float<br/>&#x2022;&nbsp;high_discharge_gradient, low_discharge_gradient, high_charge_gradient, low_charge_gradient, high_return_gradient, low_return_gradient: float |

### Ramping

Some modes accept a `ramp_params` dictionary specifying how the control handles transitions between states. Ramping
may be specified with time constants or with ramp rates:

| Ramping Type  | Parameters                                                                                                                                                                             |
|---------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Time Constant | &#x2022;&nbsp;ramp_up_time_constant: float = None<br/>&#x2022;&nbsp;ramp_down_time_constant: float = None                                                                              |
| Ramp Rate     | &#x2022;&nbsp;discharge_ramp_up_rate: float = 1000<br/>&#x2022;&nbsp;discharge_ramp_down_rate: float = 1000<br/>&#x2022;&nbsp;charge_ramp_up_rate: float = 1000<br/>&#x2022;&nbsp;charge_ramp_down_rate: float = 1000 |

## Control Evaluation Engine implementations

The active power MESA modes also have implementations in the Julia
[Control Evaluation Engine](es-control-integration.md), which the Real-Time Control Agent can run on real hardware
through the [Novel Real-Time Control](novel-real-time-control.md) mode family. This allows the same MESA mode
implementation to be evaluated in simulation with [ES-Control](https://es-control.pnnl.gov/) and then deployed on a
physical storage system.
