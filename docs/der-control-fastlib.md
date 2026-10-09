# der-control-fastlib Runtime

[Code & Installation Instructions](https://github.com/interoperability-control-modules/der-control-fastlib){ .md-button }

der-control-fastlib is the lightweight runtime on which the Interoperability Framework and Control Module
applications are deployed. It is the Interoperability Framework and Control Module distribution of the Agent Energy Management System (AEMS)
library, a fork of [aems-lib-fastapi](https://github.com/VOLTTRON/aems-lib-fastapi) from the VOLTTRON team. It
replaces the full VOLTTRON platform with two small pieces:

* **A server** (`aems-server`) built on FastAPI that provides a WebSocket message bus for agent communication, a
  REST configuration store for agent settings, and health and version endpoints.
* **A client library** (`derhost.client`) that provides an `Agent` base class with the same programming interface as a
  VOLTTRON agent: `vip.pubsub`, `vip.rpc`, `vip.config`, `RPC.export`, `Core.receiver`, and `Core.periodic`.

Each framework component and application runs as its own process, connects to the server over a WebSocket, and reads
its configuration from the server's configuration store. Because the client API mirrors VOLTTRON's, the same agent
code can be deployed on either runtime. A compatibility layer also serves legacy agents written against the monolithic
`volttron.platform` imports, including `volttron.platform.scheduling` (`cron`, `periodic`), so the
[Interoperability Service](interoperability-service.md) runs unchanged on either platform.

!!! note "Package rename"
    The Python package was renamed from `aems` to `derhost` in September 2026 and the distribution is now named
    `der-control-fastlib`. Import paths in agent code change from `aems.` to `derhost.`; the `aems-server` console
    command is unchanged.

## Installation

Requirements: Python 3.10 or higher and pip.

```shell
python -m venv ~/interoperability-framework
source ~/interoperability-framework/bin/activate
pip install git+https://github.com/interoperability-control-modules/der-control-fastlib
```

For development, clone the repository and use the `make` targets (`make dev-install`, `make test`, `make lint`,
`make format`, `make check`, `make security`, `make build`):

```shell
git clone https://github.com/interoperability-control-modules/der-control-fastlib
cd der-control-fastlib
make dev-install
```

## Starting the server

```shell
export VOLTTRON_HOME=~/.interoperability-framework
export JWT_SECRET_KEY=$(python -c "import secrets; print(secrets.token_urlsafe(32))")
aems-server --host 127.0.0.1 --port 8000
```

| Option / variable | Default                            | Description                                                                        |
|-------------------|------------------------------------|------------------------------------------------------------------------------------|
| `--host`          | `127.0.0.1`                        | Address to bind to. Use `0.0.0.0` to accept agents from other hosts.               |
| `--port`          | `8000`                             | Port to listen on.                                                                 |
| `--volttron-home` | `$VOLTTRON_HOME`                   | Base directory for runtime files.                                                  |
| `--config-dir`    | `$VOLTTRON_HOME/aems_config_store` | Directory in which the configuration store keeps agent configurations.             |
| `JWT_SECRET_KEY`  | unset                              | Secret used to sign tokens issued by `/authenticate`. The server rejects a value shorter than 32 bytes or equal to a published example, and `/authenticate` answers `503` while no key is configured. |
| `DERHOST_HOST`    | `127.0.0.1`                        | Bind address used by the `start-server` helper script.                             |

The server can also be started from Python with
`derhost.server.fastapi_message_bus.start_server(host, port, config_store_dir)`. Once running, interactive API
documentation is served at `http://<host>:<port>/docs`.

## Server API

| Endpoint                                             | Purpose                                                                                     |
|------------------------------------------------------|---------------------------------------------------------------------------------------------|
| `WS /ws/{identity}`                                  | WebSocket connection for an agent with the given identity. Carries publish, subscribe, RPC, and RPC response messages. |
| `GET /config-store/list?agent_id=`                   | List stored configurations, optionally for one agent.                                       |
| `GET /config-store/{agent_id}/{config_name}`         | Read a configuration (`?raw=true` for the raw file).                                        |
| `PUT /config-store/{agent_id}/{config_name}`         | Store a configuration (JSON body, or text with the appropriate content type).               |
| `POST /config-store/{agent_id}/{config_name}/file`   | Upload a configuration file (form field `file`; `?config_type=` overrides the type).        |
| `DELETE /config-store/{agent_id}/{config_name}`      | Delete a configuration.                                                                     |
| `GET /connections`, `GET /connections/{identity}`    | Connected agent identities, used by the container stack's readiness check.                  |
| `GET /health`, `GET /health/{agent_id}`              | Health status of all connected agents, or of one agent.                                     |
| `GET /version`                                       | Server version.                                                                             |

Agents subscribed to a configuration through `vip.config.subscribe` are notified when it is stored, updated, or
deleted, so configurations can be changed at runtime without restarting the agent.

An identity may hold one connection at a time: a second connection attempt with an already-connected identity is
refused with HTTP `409` and does not disturb the existing agent. The server sends WebSocket keepalive pings every
10 seconds with a 10-second timeout, so a dropped agent is detected and its identity freed without waiting for an
RPC to fail. Each connection may have at most 128 RPCs in flight; further RPCs are rejected with an error asking the
caller to retry, and a pending RPC never blocks an agent from reconnecting after a restart.

## Running an agent

An agent is a Python class derived from `derhost.client.agent.Agent`. The `run_agent` helper turns it into a command
line program that connects to the server:

```python
from derhost.client.agent import Agent, Core, RPC, run_agent

class Listener(Agent):
    @Core.receiver("onstart")
    def _onstart(self, sender=None, **kwargs):
        self.vip.pubsub.subscribe("devices/", self._on_message)

    def _on_message(self, peer, sender, bus, topic, headers, message):
        print(topic, message)

    @RPC.export
    def get_version(self):
        return "1.0.0"

if __name__ == "__main__":
    raise SystemExit(run_agent(Listener))
```

```shell
python listener.py --identity listener --host 127.0.0.1 --port 8000 --config listener.json
```

| Argument / variable  | Description                                                                                          |
|----------------------|------------------------------------------------------------------------------------------------------|
| `--identity`         | The agent's identity on the bus (also the `agent_id` in the configuration store). Defaults to `AGENT_VIP_IDENTITY` or the lower-cased class name. |
| `--host`, `--port`   | Address of the `aems-server`.                                                                        |
| `--config`           | JSON or YAML configuration file. Its contents are stored in the configuration store as `config` on start-up. |
| `--volttron-home`    | Runtime directory, defaulting to `$VOLTTRON_HOME`.                                                   |

Cron schedules, whether given to `Core.periodic` as a cron string or built with `cron()` from the compatibility
layer, follow the crontab convention with Sunday as day `0`. A pattern that can never match, such as February 31, or
that combines February 29 with a day-of-week restriction, is rejected when the schedule is created rather than
stalling the scheduler later.

## Containers

The repository provides a container stack under `docker/`, driven by `make` targets:

| Command                              | Effect                                                                                          |
|--------------------------------------|-------------------------------------------------------------------------------------------------|
| `make stack-up C=server`             | Build the server image and start it, waiting for `/health`.                                     |
| `make stack-status`                  | Show the compose status of the stack.                                                           |
| `make stack-check EXPECTED=a,b`      | Query `/connections` and confirm that the listed identities are connected.                      |
| `make stack-down`                    | Stop agent containers, then the server.                                                         |
| `docker/interoperability-service/build.sh` | Build the Interoperability Service image from the checkout named by `DER_AGENT_SRC`.      |
| `docker/smoke.sh`                    | Developer end-to-end test: build, start, check, and remove the stack in an isolated project.    |

The server container publishes port 8000 on host port 5410, bound to `DERHOST_PUBLISH_HOST` (default `127.0.0.1`),
and joins the external network `derhost-net` that agent containers attach to. Containers run as an unprivileged user
with all capabilities dropped. A worked deployment is given under
[Container deployment](deployment.md#container-deployment).

## Security

The server binds to `127.0.0.1` by default and does not authenticate WebSocket or REST callers. For deployments in
which agents connect from other hosts, run the server behind a reverse proxy with TLS termination and restrict access
to the bus port at the network level. Credentials and tokens are redacted from the server's log records.

## Releases

The repository uses GitHub Actions for continuous integration (tests on Python 3.10 to 3.12, formatting, linting,
security scans) and for releases. Pushing a tag of the form `v1.2.3-alpha.1` from the `develop` branch creates a
GitHub pre-release and notifies test servers; pushing a final tag `v1.2.3` from `main` creates a release and
publishes the package.
