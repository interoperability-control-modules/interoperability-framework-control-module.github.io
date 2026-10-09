# OpenFMB Integration

[Message Bus Adapter](https://github.com/interoperability-control-modules/message-bus-adapter){ .md-button }
[MQTT Protocol Proxy](https://github.com/interoperability-control-modules/lib-protocol-proxy-mqtt){ .md-button }
[NATS Protocol Proxy](https://github.com/interoperability-control-modules/lib-protocol-proxy-nats){ .md-button }
[OpenFMB Test Tool](https://github.com/interoperability-control-modules/openfmb_der){ .md-button }

Message Bus is a critical middleware component in the distributed systems architecture on various systems
and environments. Open Field Message Bus (open FMB) is an extensive framework for robust communication between
the microgrid controller and assets on the communication layer. OpenFMB was developed to reduce the complexities
of microgrid communication, such as low standards with the data structure combined with the communication protocols
towards the grid utilities. For instance, open FMB can be used to translate traditional protocols, including Modbus
and DNP3 and NATS.

OpenFMB provides the interoperable messaging layer connecting heterogeneous DER devices, controllers, and
applications. Integration of OpenFMB systems with this interoperability framework is provided by the
[Message Bus Adapters](message-bus-adapter.md) (over [MQTT](mqtt.md) or [NATS](nats.md)) and by the
[Interoperability Service](interoperability-service.md), as described in
[Integration with the Interoperability Framework](#integration-with-the-interoperability-framework) below.

## Architecture

Message buses provide asynchronous communication that performs message transfer between the senders and receivers
without a need for immediate response from the receivers as soon the sender sends the message. The components in
the message bus are independent which supports scalability and system resilience. Generally, message bus works based
on the publisher and subscriber. Open FMB is designed for three different layers, application, adapter and interface
layers. The architecture is shown in [](#openfmb-architecture).

Figure: OpenFMB Architecture {#openfmb-architecture}

![](images/openfmb-architecture-layers.png)




## Open-FMB Based Grid

The **Open-FMB Based Grid** is designed to enhance flexibility, interoperability, and efficiency in energy
management systems.

Figure: OpenFMB Key Features and Properties {#openfmb-features}

![](images/openfmb-features.png)

**Key Features:**


1. **Distributed and Decentralized Architecture**: The Open-FMB framework is built upon a distributed and decentralized architecture, promoting localized decision-making and minimizing reliance on centralized control systems. This structure allows for enhanced adaptability and rapid response to changing conditions in the energy landscape.

2. **Event-Driven Communication**: The communication model is event-driven, utilizing a **publish-subscribe** mechanism that facilitates asynchronous message exchanges. This model allows devices to send and receive updates efficiently, ensuring real-time responsiveness.

3. **Interoperability**: The framework emphasizes **interoperability** among diverse devices and systems by leveraging **open standards** such as DDS (Data Distribution Service) and MQTT (Message Queuing Telemetry Transport). This ensures that equipment from various manufacturers can work together seamlessly.

4. **Scalability**: Open-FMB is designed for **high scalability**, allowing the system to grow and adapt without compromising performance. This is crucial in accommodating the increasing number of edge devices and data sources as the energy grid evolves.

5. **Reliability and Peer-to-Peer Resilience**: With a focus on **reliability**, the Open-FMB architecture provides mechanisms for **secure messaging** between devices. The peer-to-peer structure enhances resilience, allowing for uninterrupted communication even if individual components fail.

6. **Support for Edge Devices**: The architecture supports a wide range of **edge devices**, facilitating the integration of sensors, controllers, and other local elements that can actively participate in energy management and monitoring.

7. **Distributed Decision Making**: By enabling **distributed decision-making**, the Open-FMB Grid supports more efficient and localized responses to grid conditions, allowing for quicker adjustments based on real-time data.

## Integration with the Interoperability Framework

The framework extends OpenFMB from basic message exchange toward scalable multi-device coordination.
As shown in [](#openfmb-integration), the Interoperability Service connects to OpenFMB buses through message bus
adapters while continuing to talk directly to devices through drivers:

* **[Message bus adapters](message-bus-adapter.md)** connect the internal framework message bus with external OpenFMB
  buses over [NATS](nats.md) and [MQTT](mqtt.md).
  An OpenFMB device adapter on the external bus exposes SunSpec, MESA (DNP3), and IEEE 2030.5 DER devices as OpenFMB
  nodes, and OpenFMB applications on that bus can interact with the framework.
* **Device drivers** enable direct point-to-point communication with controllers and DER devices over Modbus, DNP3,
  and HTTP (IEEE 2030.5).
* **Identifier mapping and data-model transformations**, provided by the mapping and transform registries of the
  Interoperability Service, enable communication across all supported protocols. OpenFMB is a first-class format in
  the transform registry: the `openfmb.ess` and `openfmb.solar` formats carry ESS and solar profiles in the protobuf
  JSON form that OpenFMB adapters publish, and
  [bundled transforms](interoperability-service.md#bundled-transforms) map them to and from IEC 61850-7-420, through
  which they reach SunSpec, IEEE 1815.2, IEEE 2030.5, and the IEEE 1547.1 view. The service also includes generated
  [pydantic models of all 67 OpenFMB profiles](interoperability-service.md#openfmb-and-standards-data-models) with
  profile builders for constructing complete OpenFMB profiles.
* Both **publish/subscribe** and **direct device communication** are supported.

Figure: OpenFMB integration. Message bus adapters (MQTT, NATS) connect the Interoperability Service to an external
OpenFMB bus, while device drivers provide direct access to SunSpec, MESA, and IEEE 2030.5 devices.
{#openfmb-integration}

![](images/openfmb-integration.png)

Code for the OpenFMB integration components:

* [message-bus-adapter](https://github.com/interoperability-control-modules/message-bus-adapter)
* [lib-protocol-proxy-nats](https://github.com/interoperability-control-modules/lib-protocol-proxy-nats)
* [lib-protocol-proxy-mqtt](https://github.com/interoperability-control-modules/lib-protocol-proxy-mqtt)
* [openfmb_der](https://github.com/interoperability-control-modules/openfmb_der) (OpenFMB test tool)
