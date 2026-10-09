# Test Cases

The communication and control framework has been tested on the PNNL campus using a 125kW/250kWh BESS and a building
with a 150kW peak load. Tests involving energy arbitrage, demand charge reduction and MESA charge/ discharge modes
are further discussed in
[Interoperable Energy Storage Control and Communication Framework Development](https://ieeexplore.ieee.org/document/10891219).


# Experimentation Results with VOLTTRON

In the VOLTTRON platform, the Battery Energy Storage Systems (BESS) within the grid is integrated using modular agents for efficiency and cost-effectiveness. Multiple agents such as the [Forecaster](forecaster-agent.md), [Grid Signals](grid-signals.md), [Real-Time Control](rt-control.md) (MESA Charge/Discharge Power mode) and [Scheduler](scheduler.md) are the data source for the system with the help of external data sources include CO2 intensity, energy generation breakdown and electricity prices from APIs to indicate BESS operations.


## Key Features

| Feature                  | Description                                                                                   |
|--------------------------|-----------------------------------------------------------------------------------------------|
| **Forecast Inputs**      | Day-ahead forecasts of CO2 intensity, energy generation mix, and electricity prices.          |
| **Optimization Process** | Provides outdoor temperature and load forecasts for scheduling BESS operations.               |
| **Optimization Objectives** | Focuses on minimizing electricity expenses using forecasted price signals for a 24-hour operation plan. |
| **Constraints**          | Ensures BESS power limits and State of Charge (SoC) boundaries are maintained.                |
| **Power Management**     | Utilizes price signals for efficient battery charging and discharging.                        |
| **Modular Integration**  | VOLTTRON agents collaborate to provide dynamic and efficient energy management.               |

Figure: MESA Charge/Discharge mode implementation, showing the Scheduler agent's 24-hour battery operation plan based
on price signals to minimize costs and reduce peak power. {#mesa-charge-discharge}

![](image.png)


This implementation highlights how predictive modeling and real-time data enhance energy management and grid service delivery through VOLTTRON, demonstrating adaptive, efficient and cost-effective operations.

# Industry Engagement and Ongoing Testing

To validate the interoperability framework against real vendor equipment, PNNL is engaging key industry vendors,
including SEL, Eaton, WAGO, Leidos, and Schneider Electric, and has established a dedicated interoperability test
environment at PNNL. The test setup, shown in [](#testing-setup), hosts the interoperability framework on a WAGO
processor and connects it to:

* **Device gateways with integrated end devices**: Eaton grid edge devices (e.g., EdgeAP / Grid Edge Manager),
  SEL RTAC (e.g., RTAC 3555/3505), and WAGO grid edge controllers, each with PV, BESS, inverter, meter, feeder, relay,
  or load end devices behind them, communicating over standard protocols (Modbus, DNP3, IEC 61850).
* **Simulated end devices**: software models of BESS, PV, wind, flexible load, and meters, reached directly from the
  framework; see [Simulated end devices](#simulated-end-devices) below.

Supported protocols in the test environment include Modbus (SunSpec), DNP3 (IEEE 1815.2), IEC 61850, IEEE 2030.5 (SEP),
OpenFMB/MESA, and vendor-specific protocols via drivers. The goals of this effort are to:

* Validate the interoperability framework using real vendor devices, including gateways, controllers, and DER
  interfaces from Eaton, SEL, and WAGO.
* Demonstrate interoperability across multiple protocols and data models, including OpenFMB, IEEE 1547 /
  IEC 61850-7-420, and utility integration interfaces.
* Conduct structured and repeatable interoperability testing across multiple use cases, using vendor feedback to
  refine data models, message mappings, framework capabilities, and real-time control workflows.

Figure: Path to a protocol-agnostic, interoperable DER ecosystem: vendor engagement, device integration,
interoperability testing, and framework refinement. {#industry-engagement}

![](images/industry-engagement.png)

Figure: Ongoing interoperability test setup at PNNL. Path 1 reaches end devices through vendor gateways (Eaton,
SEL RTAC, WAGO controller); Path 2 connects the framework directly to simulated end devices. {#testing-setup}

![](images/testing-setup.png)

## Simulated end devices

[Code & Installation Instructions](https://github.com/interoperability-control-modules/simulated-end-devices){ .md-button }

The `simulated-end-devices` package provides the Path 2 devices of [](#testing-setup): software models of DERs that
expose the standard protocols the framework speaks, so that the Interoperability Service, drivers, and control
applications can be exercised without hardware. One site configuration runs six devices behind a shared simulation
clock and shared grid conditions:

| Device id    | Model                              | Protocol                                             | Default port |
|--------------|------------------------------------|------------------------------------------------------|--------------|
| `bess_modbus`| 250 kWh / 125 kW battery           | SunSpec Modbus TCP (models 1, 701, 702, 704, 713)    | 5021         |
| `bess_dnp3`  | 500 kWh / 250 kW battery           | IEEE 1815.2 / MESA-DER (DNP3 outstation)             | 20000        |
| `bess_2030_5`| 100 kWh / 50 kW battery            | IEEE 2030.5 (HTTP, `application/sep+xml`)            | 8080         |
| `pv`         | 100 kW DC / 90 kW AC PV array      | SunSpec Modbus TCP (models 1, 701, 702, 704)         | 5022         |
| `wind`       | 50 kW wind turbine                 | SunSpec Modbus TCP (models 1, 701, 702, 704)         | 5023         |
| `meter`      | Site meter at the point of common coupling | SunSpec Modbus TCP (models 1, 203)           | 5024         |

The register maps, point indices, and resource trees follow the respective standards, so the transforms bundled with
the [Interoperability Service](interoperability-service.md#bundled-transforms) apply without site-specific rules:

* **SunSpec Modbus** devices place the SunSpec model chain at holding register 40000. Model 704 is writable, so an
  active-power limit (`WMaxLimPctEna`, `WMaxLimPct`) or a battery power set point (`WSetEna`, `WSetMod`, `WSet`) takes
  effect on the next simulation step.
* The **IEEE 1815.2** battery accepts MESA-DER controls such as `DWGC_Mod` with `DWGC_GnWPctSpt`, `DAGC_Mod` with
  `DAGC_WSpt`, and `DGEN_PrmConn` / `DGEN_PrmDscon`. It runs as a real OpenDNP3 outstation when `dnp3-python` is
  installed (Linux); otherwise, or with `json_fallback: true`, the same point database is served as newline-delimited
  JSON over TCP.
* The **IEEE 2030.5** battery serves the resource tree (`/dcap`, `/edev/1`, `/edev/1/der/1`, `/derp/1/derc`,
  `/derp/1/dderc`, `/mup/1`, ...) and accepts posted `DERControl` objects with `opModTargetW`, `opModFixedW`,
  `opModMaxLimW`, and `opModConnect`. It does not implement TLS or certificate-based device identity, which a
  production IEEE 2030.5 deployment requires, and is therefore for testing only.

Install and run the site in real time, or accelerated:

```shell
git clone https://github.com/interoperability-control-modules/simulated-end-devices
cd simulated-end-devices
python -m venv .venv && source .venv/bin/activate
pip install .                                                # add ".[dnp3]" on Linux for a real DNP3 outstation
simulated-devices run --config configs/site.yaml             # real time
simulated-devices run --config configs/site.yaml --speed 60  # one simulated minute per second
simulated-devices points --config configs/site.yaml          # dump register, point, and resource maps
```

A `docker compose up --build` in the repository starts the same site in containers. To connect the framework, point
its SunSpec, DNP3, and IEEE 2030.5 drivers at the ports above and load `configs/interoperability-mappings.json` into
the Interoperability Service; it registers the three batteries as canonical resources and adds IEC 61850 aliases
`pnnl-sim/ess/1` to `pnnl-sim/ess/3` for them.
