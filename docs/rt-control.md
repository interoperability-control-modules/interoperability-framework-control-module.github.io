# Real-Time Control Agent

[Code & Installation Instructions](https://github.com/interoperability-control-modules/realtime-control-agent){ .md-button }

The Real-Time (RT) Control Agent provides a framework for actuating one or more control algorithms
on an energy storage system. The RTControl framework involves the use of three abstract class types:
EnergyStorageSystem, ControlMode, and UseCase.

For each class there are several built-in subclasses, but user defined classes may also be configured and used.
The built-in control modes fall into two families, each described on its own page:

* [MESA Modes](mesa-modes.md): standards-aligned implementations of the MESA-ESS active power, reactive power, and
  emergency modes, implemented natively in Python.
* [Novel Real-Time Control](novel-real-time-control.md): PNNL-developed control functions (Adaptive Moving Average
  Control, PID, rule-based control) that run the algorithms of the Control Evaluation Engine on real hardware. See
  [ES Control Integration](es-control-integration.md).

### EnergyStorageSystem

ESS classes abstract an energy storage system into a standard interface for use by control modes.
The base class allows configuration of points on which state of charge and power
can be monitored and (in the case of power) commanded.
Currently two built-in ESS classes are shown in [](#storage-systems)

Table: Energy Storage Systems Implemented in the Real-Time Control Agent {#storage-systems}

| Storage System    | Description                                                                                                                                                                                                                                                                                                          |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| FakeESS           | Intended only for testing, commands sent to the FakeESS update in memory representations of the state of charge and power.                                                                                                                                                                                           |
| SebBESS           | This class is designed to communicate with a simple battery system which has a single control point in Watts. Other storage systems with a similar interface may be configured using the same class, or it may be used as a model for writing a custom class to command a system with a different control structure. |

### UseCase

UseCases are information services intended to collect data necessary to actualize some goal with the storage system.
The UseCase ingests data from elsewhere in the VOLTTRON ecosystem, performs any necessary model calculations or
transformations, and makes it available to control modes through properties. Eight built-in Use Cases are available,
as shown in [](#use-cases) (It should be noted that some modelled values are provided only as stubs.
These may be implemented by users who subclass these with their own model.)

Table: Use Cases Implemented in the Real-Time Control Agent {#use-cases}

| Use Case               | Description  | Inputs                                        |
|------------------------|--|-----------------------------------------------------------|
| Energy Arbitrage       | Energy arbitrage refers to the operation of energy storage that discharges when the electricity prices are high and charges when the prices are low. Since this type of energy storage operation reduces the net system load during peak hours and increases the load during off-peak hours, it is also often referred to as load leveling or load shifting. Energy arbitrage can be performed in both a vertically integrated system and in wholesale electricity markets. The economic reward is the price or cost differential between charging and discharging electrical energy minus the cost of losses during the full charging/discharging cycle. | &#x2022;&nbsp;actual_price<br/>&#x2022;&nbsp;forecast_price&nbsp;(stub)  |
| Frequency Response     | The ESS is configured to independently respond to excursions from nominal frequency by altering its power output or input. The parameters by which the control will be actuated are set using vertices or gradients in a Frequency-Watt curve. | &#x2022;&nbsp;metered_frequency                                         |
| Generation Following   | Generation is fully or partially countered by using the ESS to absorb energy (charging) when metered generation rises beyond a configured threshold. This may be used to prevent export to the grid or as a mechanism for charging the ESS when local generation is high. | &#x2022;&nbsp;forecast_power&nbsp;(stub)<br/> &#x2022;&nbsp;realtime_power                  |
| Load Following         | Load is fully or partially countered by discharging the ESS when metered load rises above some threshold. | &#x2022; forecast_power (stub)<br>&#x2022;&nbsp;realtime_power                   |
| Peak Limiting          | Metered load, beyond some configured threshold, is fully countered by discharing the ESS until load drops below this threshold again. This can be used as a mechanism to avoid capacity charges. | &#x2022;&nbsp;realtime_power                                            |
| Regulation             | The electric power system must maintain a near-real-time balance between generation and load. Balancing generation and load instantaneously and continuously is difficult because loads and generation are constantly fluctuating. Frequency regulation, also known as automatic frequency restoration reserve (aFRR) in continental Europe, are required to continuously balance generation and load under normal operating conditions. Traditionally, the majority of frequency regulation capability has been provided by specially equipped generators. As technologies evolve, new types of flexibility resources emerge, such as ESSs. | &#x2022;&nbsp;agc_signal<br/>&#x2022;&nbsp;price&nbsp;(stub)<br/>&#x2022;&nbsp;performance_score&nbsp;(stub)  |
| Variability Mitigation | A power smoothing algorithm reduces power fluctuations from renewable energy sources or volatile loads. It manages energy storage systems to store excess power during high generation or low demand, and release stored power during low generation or high demand. It employs real-time monitoring and control systems to adjust power in response to changing conditions. The algorithm enhances stability and reliability of renewable energy integration and optimizes energy storage utilization. Variability (a.k.a. ramp-rate, volatility, or intermittency) is defined as an instantaneous change in a load or source power, e.g., rapid changes in solar output power due to an intermittent cloud cover. | &#x2022;&nbsp;forecast_power&nbsp;(stub)<br/>&#x2022;&nbsp;metered_power |
| Voltage Control        | Provides the measured voltage at the reference point for the reactive power and voltage-based MESA modes (Volt-VAR, Volt-Watt, dynamic reactive current support, voltage ride-through). | &#x2022;&nbsp;metered_voltage |

### Mode

Control Modes contain the implementation of an algorithm for actuating the storage system.
These may, optionally, ingest data from UseCases. They control the storage hardware through
the interface of EnergyStorageSystem classes. The built-in modes are listed in [](#control-mode-families) and
described in detail on the [MESA Modes](mesa-modes.md) and [Novel Real-Time Control](novel-real-time-control.md)
pages.

Table: Control mode families implemented in the Real-Time Control Agent {#control-mode-families}

| Family                                              | Modes                                                                                                                                                      |
|-----------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [MESA active power modes](mesa-modes.md)            | Active Power Limit, Active Power Response, Active Power Smoothing, AGC, Charge/Discharge Storage, Frequency-Watt, Volt-Watt                                 |
| [MESA reactive power modes](mesa-modes.md)          | Constant VAR, Fixed Power Factor, Power Factor Correction, Volt-VAR, Watt-VAR, Dynamic Reactive Current Support                                            |
| [MESA emergency modes](mesa-modes.md)               | Frequency Ride-Through, Voltage Ride-Through                                                                                                               |
| [Novel real-time control](novel-real-time-control.md) | Adaptive Moving Average Control (AMAC), PID, Rule Based, and Control Evaluation Engine implementations of the active power MESA modes                     |

## Configuration

The agent is configured using a JSON file which accepts the following parameters:

| Parameter                   | Description                                                                                                                  |
|-----------------------------|------------------------------------------------------------------------------------------------------------------------------|
| `ctrl_eval_engine_app_path` | The filesystem path to the app directory of the ctrl-eval-engine repository. Required only for [novel modes](novel-real-time-control.md). |
| `julia_path`                | Path to the Julia executable used by the novel modes (default `/usr/bin/julia`).                                             |
| `resolution`                | The smallest time interval considered by the control.                                                                        |
| `ess`                       | A configuration dictionary for the storage system.                                                                           |
| `use_cases`                 | A list of configuration dictionaries, one for each use case.                                                                 |
| `modes`                     | A list of configuration dictionaries, one for each control mode in use.                                                      |

Each object in `ess`, `use_cases`, and `modes` accepts `class_name` (the class to instantiate) and, for custom classes,
`module_name` (the module in which the class is found). Additional parameters depend on the class; the parameters of
each built-in mode are listed on the [MESA Modes](mesa-modes.md) and [Novel Real-Time Control](novel-real-time-control.md)
pages.

### ESS settings (`ess`)

| Parameter             | Description                                                                     |
|-----------------------|---------------------------------------------------------------------------------|
| `ess_topic`           | A VOLTTRON topic which will be monitored for publishes from the storage device. |
| `soc_point`           | The point name to read state of charge from publishes on the ess_topic.         |
| `power_reading_point` | The point name to read power from publishes on the ess_topic.                   |
| `power_command_topic` | A VOLTTRON topic which will be used to command power set points.                |
| `power_command_point` | The point name to write set point commands on the power_command_topic.          |
| `actuator_vip`        | The VOLTTRON vip-identity of the agent being used to actuate the ESS.           |
| `actuation_method`    | The method name used to command the actuator agent over RPC.                    |
| `actuation_kwargs`    | Any keyword arguments to be provided to the actuator agent.                     |
| `rounding_precision`  | The number of decimal places to which to round values when commanding power.    |

### Use case settings (`use_cases`)

Each use case data point is configured with a pair of keys, `<identifier>_topic` and `<identifier>_point`, naming the
VOLTTRON topic to subscribe to and the point within the received message. For example, the Energy Arbitrage use case
uses `actual_price_topic` and `actual_price_point`.

| Use Case               | Identifier          | Description                           |
|------------------------|---------------------|---------------------------------------|
| Energy Arbitrage       | `actual_price`      | The current price of energy.          |
| Frequency Response     | `metered_frequency` | Frequency at the reference meter.     |
| Generation Following   | `realtime_power`    | Power at the reference meter.         |
| Load Following         | `realtime_power`    | Power at the reference meter.         |
| Peak Limiting          | `realtime_power`    | Power at the reference meter.         |
| Regulation             | `agc_signal`        | The command from the system operator. |
| Variability Mitigation | `metered_power`     | Power at the reference meter.         |
| Voltage Control        | `metered_voltage`   | Voltage at the reference meter.       |

## Integration with the Interoperability Service

The RT Control Agent is integrated with the [Interoperability Service](interoperability-service.md) and OpenFMB.
Publish/subscribe messaging provides normalized device data to control applications: the RT Control Agent consumes
the published data structures and issues control commands back through the interoperability framework, regardless of
the native protocol of each device. This enables coordinated actuation of multiple heterogeneous DER/ESS devices using
MESA control modes, PNNL-developed control functions, and user-defined control algorithms, with both scheduling and
real-time control.

## Installation

Before installing, VOLTTRON should be installed and running and its virtual environment should be active. The
control modes also import on hosts without a VOLTTRON distribution, such as the
[der-control-fastlib runtime](der-control-fastlib.md), where the compatibility layer supplies the platform imports.

```shell
git clone https://github.com/interoperability-control-modules/realtime-control-agent
vctl install ./realtime-control-agent --vip-identity der.rtcontrol --tag rtcontrol --start
vctl config store der.rtcontrol config path/to/config.json
```

The [novel modes](novel-real-time-control.md) additionally require Julia and the Control Evaluation Engine; see
[ES Control Integration](es-control-integration.md).
