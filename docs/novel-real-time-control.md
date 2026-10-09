# Novel Real-Time Control

[Code & Installation Instructions](https://github.com/interoperability-control-modules/realtime-control-agent){ .md-button }

In addition to the standards-defined [MESA Modes](mesa-modes.md), the [Real-Time Control Agent](rt-control.md)
provides a family of PNNL-developed control functions. These are implemented in the Julia
[Control Evaluation Engine](es-control-integration.md), the shared backend of the
[ES-Control](https://es-control.pnnl.gov/) web tool, and are exposed to the agent through a common `ESControlMode`
base class. Because the same algorithm code runs in the ES-Control simulator and on real hardware, a control strategy
can be developed and evaluated in simulation and then actuated on a physical storage system without re-implementation.

## Control functions

Table: Novel control modes implemented through the Control Evaluation Engine {#novel-modes}

| Control Mode                            | Description                                                                                                                                                                                                                                                                                                                                                    | Parameters                                                             |
|-----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| Adaptive Moving Average Control (AMAC)  | Similar to the MESA Active Power Smoothing mode, but using an algorithm which self-optimizes the window of the moving average filter: a longer window (more aggressive smoothing) is used when variability is high and a shorter window (which utilizes less of the storage resource) when variability is low. Pairs with the Variability Mitigation use case. |                                                                        |
| PID                                     | Uses a proportional-integral-derivative loop to maintain power targets at the reference meter in response to changes in load or generation.                                                                                                                                                                                                                    | &#x2022;&nbsp;resolution<br/>&#x2022;&nbsp;kp<br/>&#x2022;&nbsp;ti<br/>&#x2022;&nbsp;td |
| Rule Based                              | Modifies the battery set point from the day-ahead plan produced by the [Scheduler](scheduler.md) in real time, according to rules that keep the metered power within a configured bound of the plan.                                                                                                                                                             | &#x2022;&nbsp;bound: float                                             |

The Control Evaluation Engine also provides implementations of the active power MESA modes (Active Power Limit,
Active Power Smoothing, AGC, Charge/Discharge Storage, Frequency-Watt, and Volt-Watt). These can be selected in
place of the native Python implementations when it is desirable to run exactly the algorithm that was evaluated in
ES-Control.

## How the modes run

Each novel mode is a thin Python wrapper around a controller struct in the Control Evaluation Engine:

1. On start-up the mode loads Julia through PyJulia, using the `julia_path` and `ctrl_eval_engine_app_path`
   configured for the [Real-Time Control Agent](rt-control.md), and activates the engine's project.
2. On every control interval the mode builds an engine representation of the current storage system from the
   EnergyStorageSystem specifications and states, converts the active Use Cases to their engine equivalents, and
   passes the current schedule period from the Scheduler.
3. The engine's controller computes the set point, which the agent then applies to the hardware through the
   EnergyStorageSystem interface, exactly as for a native MESA mode.

Native MESA modes may also fall back to their engine implementation when configured to do so, and the Julia runtime is
imported lazily, so deployments that use only native modes do not require Julia.

## Configuration

Novel modes are configured as entries of the `modes` list, like any other mode. The agent-level parameters
`ctrl_eval_engine_app_path` and `julia_path` are passed to each mode automatically:

```json
{
  "ctrl_eval_engine_app_path": "/home/volttron/ctrl-eval-engine-app",
  "julia_path": "/home/volttron/julia-1.10.4/bin/julia",
  "resolution": 300,
  "ess": { "class_name": "SebBESS", "ess_topic": "devices/PNNL/SEB/BESS/all", "...": "..." },
  "use_cases": [
    { "class_name": "PeakLimiting", "realtime_power_topic": "devices/PNNL/SEB/METER/all",
      "realtime_power_point": "WholeBuildingPower" }
  ],
  "modes": [
    { "class_name": "PID", "module_name": "rt_control.modes.novel.pid",
      "resolution": 300, "kp": 0.8, "ti": 600, "td": 0 }
  ]
}
```

## Requirements

* A dynamically linked Python environment (required by PyJulia), Julia, and the Control Evaluation Engine app
  directory. Installation steps are given on the [ES Control Integration](es-control-integration.md) page and in the
  [ctrl-eval-engine](https://github.com/interoperability-control-modules/ctrl-eval-engine) repository.
