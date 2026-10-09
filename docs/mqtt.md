# MQTT Protocol Proxy

[Code & Installation Instructions](https://github.com/interoperability-control-modules/lib-protocol-proxy-mqtt){ .md-button }

The MQTT Protocol Proxy connects the [Message Bus Adapter](message-bus-adapter.md) to an MQTT broker. MQTT (Message
Queuing Telemetry Transport) is one of the transports commonly used by OpenFMB deployments, so this proxy is the usual
route for exchanging data with OpenFMB devices and applications; see [OpenFMB Integration](openfmb.md).

The proxy is implemented in the `protocol-proxy-mqtt` Python package on top of the `protocol-proxy` library
(version 2.0.0rc2 or later) and the [paho-mqtt](https://pypi.org/project/paho-mqtt/) client. It runs as a separate
process that is launched and supervised by the Message Bus Adapter's proxy manager, so a fault in the broker
connection cannot take down the agent. The manager starts the process through the generic launcher,
`python -m protocol_proxy.proxy --gevent <module>:<class>`, which applies gevent monkey-patching before the proxy
module is imported; the proxy module itself is no longer runnable directly.

## Behavior

* On start-up the proxy connects to the configured broker and registers itself with the adapter.
* When the adapter requests a subscription, the proxy subscribes to the requested topics on the broker.
  Subscriptions are re-established automatically whenever the broker connection is (re)made.
* Each message received from the broker is forwarded to the adapter with its topic, message id, QoS, payload, and
  timestamp. The adapter resolves the topic through the [Interoperability Service](interoperability-service.md)
  and publishes the transformed payload on the local bus.
* MQTT topics use `/` as the segment delimiter, which the adapter uses when resolving topics to Uniquely Addressable
  Identifiers.

## Configuration

The proxy is configured through the `adapters` entries of the Message Bus Adapter configuration when `bus_type` is
`mqtt`:

| Parameter      | Required | Type    | Default | Description                                                                                   |
|----------------|----------|---------|---------|-----------------------------------------------------------------------------------------------|
| `host`         | true     | string  |         | Address of the MQTT broker.                                                                   |
| `port`         | false    | integer | `1883`  | Port of the MQTT broker.                                                                      |
| `keepalive`    | false    | integer | `60`    | Maximum period in seconds between communications with the broker. Controls the ping interval when no other messages are exchanged. |
| `bind_address` | false    | string  | `""`    | Local network interface to bind the client to.                                                |
| `bind_port`    | false    | integer | `0`     | Local port to bind the client to. `0` lets the operating system choose.                       |
| `client_id`    | false    | string  |         | MQTT client identifier; generated when omitted.                                               |
| `username`, `password` | false | string |      | Broker credentials.                                                                           |
| `tls`          | false    | object  |         | TLS settings passed to the paho-mqtt client.                                                  |
| `protocol`     | false    | string  |         | `MQTTv31`, `MQTTv311`, or `MQTTv5`.                                                           |
| `qos`          | false    | integer |         | Quality of service level for subscriptions and publications.                                  |
| `reconnect_min_delay`, `reconnect_max_delay` | false | integer | | Bounds, in seconds, of the reconnect back-off.                                    |

The same options are accepted on the command line (`--host`, `--port`, `--keepalive`, `--bind-address`,
`--bind-port`) when the proxy is launched directly for testing.

## Installation

```shell
pip install git+https://github.com/interoperability-control-modules/lib-protocol-proxy-mqtt
```

Install the package into the same virtual environment as the [Message Bus Adapter](message-bus-adapter.md). The
adapter discovers the proxy automatically when `bus_type` is set to `mqtt`.
