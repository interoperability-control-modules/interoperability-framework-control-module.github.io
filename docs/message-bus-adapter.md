# Message Bus Adapters

[Code & Installation Instructions](https://github.com/interoperability-control-modules/message-bus-adapter){ .md-button }

The Message Bus Adapter is a VOLTTRON agent that relays data between the internal framework message bus and a
foreign message bus, such as an OpenFMB bus running on [MQTT](mqtt.md) or [NATS](nats.md). It is the component that
allows the [Interoperability Service](interoperability-service.md) and the control agents to exchange data with
devices and applications that live on an external publish/subscribe bus, as shown on the
[OpenFMB Integration](openfmb.md) page.

The adapter itself is bus-agnostic. The connection to a specific bus technology is provided by a **protocol proxy**,
a separate process managed by the adapter through the `protocol-proxy` library. Two proxy implementations are
currently available:

| Bus  | Protocol proxy                                                                          | Client library |
|------|-----------------------------------------------------------------------------------------|----------------|
| MQTT | [lib-protocol-proxy-mqtt](https://github.com/interoperability-control-modules/lib-protocol-proxy-mqtt) | paho-mqtt      |
| NATS | [lib-protocol-proxy-nats](https://github.com/interoperability-control-modules/lib-protocol-proxy-nats) | nats-py        |

## How it works

The adapter starts a protocol proxy manager for the configured `bus_type` and launches one proxy process per remote
broker. Messages then flow in both directions:

* **Remote to local.** When a proxy receives a message from the remote bus, it forwards it to the adapter. The adapter
  asks the Interoperability Service to resolve the remote topic (split using the bus's own delimiter, `/` for MQTT
  and `.` for NATS) to a canonical resource. The resource definition includes the transform needed to convert the
  remote payload into the local data format. The adapter applies the transform and publishes the result on the
  VOLTTRON message bus at the resource's local topic.
* **Local to remote.** When a remote peer requests a subscription to a local topic, the adapter resolves the topic in
  the same way, subscribes to the canonical publication topic on the VOLTTRON bus, and forwards each transformed
  message back to the requesting proxy, which publishes it on the remote bus.

Because all topic resolution and payload transformation is delegated to the Interoperability Service, the adapter
needs no protocol-specific mapping logic of its own. The service's
[bundled transforms](interoperability-service.md#bundled-transforms) among IEC 61850, IEEE 1815.2, IEEE 2030.5,
IEEE 1547.1, SunSpec, and OpenFMB are available immediately; adding a mapping or transform to the service is
sufficient to expose a new resource across the bus boundary.

## Interface

The adapter runs with the VIP identity `platform.bus_adapter` and exports the following RPC methods for local agents:

| Method                                           | Description                                                                                                                     |
|--------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `list_remotes()`                                 | Returns the `unique_remote_id` of every configured remote bus.                                                                   |
| `subscribe(unique_remote_id, topics)`            | Instructs the proxy identified by `unique_remote_id` to subscribe to the given topics (or subjects) on the remote bus. Messages received on those topics are relayed to the local bus as described above. |
| `publish(unique_remote_id, topic, payload)`      | Publishes a JSON-serializable payload to a topic on the remote bus through the identified proxy.                                  |

A `unique_remote_id` is `[bus_type, name]` when the adapter entry has a `name`, otherwise `[bus_type, host, port]`
for MQTT and `[bus_type, servers]` for NATS. Only remotes present in the configuration can be addressed.

## Configuration

The adapter is configured with a JSON document specifying the bus type and a list of remote bus connections. For an
MQTT bus:

```json
{
  "bus_type": "mqtt",
  "proxy_registration_timeout": 30,
  "adapters": [
    {"name": "site_broker", "host": "broker.example.org", "port": 1883},
    {"host": "10.0.0.5", "port": 8883, "keepalive": 30}
  ]
}
```

| Parameter                    | Required | Type   | Default | Description                                                                                         |
|------------------------------|----------|--------|---------|-----------------------------------------------------------------------------------------------------|
| `bus_type`                   | true     | string |         | Type of the foreign bus, `mqtt` or `nats`. Resolved to the plugin `protocol_proxy.protocol.<bus_type>`. |
| `adapters`                   | true     | list   |         | One entry per remote broker. Every field except `name` is passed to the proxy process as a command line argument, so the fields must match the proxy's parameters; see [MQTT](mqtt.md) and [NATS](nats.md). Fields left unset are not sent, so the proxy's own defaults apply. |
| `adapters[].name`            | false    | string |         | Optional handle used to form the `unique_remote_id`.                                                |
| `proxy_registration_timeout` | false    | number | `30`    | Seconds to wait for a newly launched proxy process to register with the adapter.                    |

A proxy is launched for every configured adapter when the configuration is loaded, and the adapter stays idle with a
warning until a `bus_type` is configured. Changing `bus_type` at runtime is not supported; restart the agent instead.

Store the configuration in the VOLTTRON configuration store:

```shell
vctl config store platform.bus_adapter config path/to/config.json
```

## Requirements

* Python >= 3.10
* Modular Eclipse VOLTTRON (`volttron-core` >= 2.0)
* `protocol-proxy` >= 2.0.0rc1 and at least one protocol proxy implementation ([MQTT](mqtt.md) or [NATS](nats.md))
* A running [Interoperability Service](interoperability-service.md) for topic resolution and payload transformation

## Installation

Before installing, VOLTTRON should be installed and running and its virtual environment should be active.
Install the protocol proxy for the bus you intend to use, then install the adapter:

```shell
pip install git+https://github.com/interoperability-control-modules/lib-protocol-proxy-mqtt   # or lib-protocol-proxy-nats
git clone https://github.com/interoperability-control-modules/message-bus-adapter
vctl install ./message-bus-adapter --vip-identity platform.bus_adapter --tag bus_adapter --start
vctl config store platform.bus_adapter config path/to/config.json
```

View the status of the installed agent:

```shell
vctl status
```
