# Building https://interoperability-control-modules.github.io/interoperability-framework-control-module.github.io

This repository contains documentation using MkDocs for [interoperability-framework-control-module.github.io](https://interoperability-control-modules.github.io/interoperability-framework-control-module.github.io). Follow these steps to set up and build the MkDocs documentation locally.

## Prerequisites

- Python 3.x installed
- Pip package manager

## Installation Steps

### 1. Clone the Repository

Clone this repository to your local machine:

```bash
git clone https://github.com/interoperability-control-modules/interoperability-framework-control-module.github.io
cd your-repo
```

### 2. Install Requirements

Install requirements using pip:

```bash
pip install -r requirements.txt
```

### 3. Preview the Documentation Locally

To view the documentation locally:

```bash
mkdocs serve
```

This command starts a development server and provides a local preview of the documentation. By default, it will be available at http://127.0.0.1:8000/.

### 4. Make Changes

Create a new branch/fork for changes that you make.  Edit the Markdown files in the docs directory to update the documentation content. Changes made to these files will be reflected in the local preview automatically.
Create a pull request to the main branch for inclusion in the site.

## Site Structure

The documentation pages live in the `docs` directory and are organized by `mkdocs.yml`:

| Page                                    | Content                                                                                  |
|-----------------------------------------|------------------------------------------------------------------------------------------|
| `docs/index.md`                         | Introduction, background, problem definition, and proposed solution.                     |
| `docs/interoperability-framework.md`    | Overview of the framework components.                                                    |
| `docs/interoperability-service.md`      | The Interoperability Service (protocol-agnostic mapping and transform service).          |
| `docs/message-bus-adapter.md` | Message Bus Adapter agent relaying data to external (OpenFMB) buses. |
| `docs/mqtt.md`, `docs/nats.md` | MQTT and NATS protocol proxies used by the Message Bus Adapter. |
| `docs/rt-control.md`, `docs/mesa-modes.md`, `docs/novel-real-time-control.md` | Real-Time Control Agent, MESA modes, and novel control functions. |
| `docs/scheduler.md`, `docs/es-control-integration.md` | Scheduler and ES-Control / Control Evaluation Engine integration. |
| `docs/grid-signals.md`, `docs/forecaster-agent.md` | Grid Signals and Forecaster agents. |
| `docs/openfmb.md` | OpenFMB Integration with the framework via message bus adapters. |
| `docs/der-control-fastlib.md`, `docs/deployment.md` | The der-control-fastlib runtime and the deployment guide. |
| `docs/test-cases.md`                    | Test cases, experimentation results, and the ongoing industry test setup.                |

Figures are stored in `docs/images`. The source presentation and figures used on the site are maintained in the
`presentation` folder of the `interoperability-framework` workspace.
