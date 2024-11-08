# [MQTT](https://mqtt.org/) Discovery Handler

## MQTT and its use

MQTT is a simple message bus with a broad use in IoT devices, such device will usually connect to an existing MQTT broker on the network.
This allows reuse of the same broker for multiple devices.

Let's quickly introduce some key concepts of MQTT:

- An MQTT message consists of a slash (`/`) separated UTF-8 topic and a completely free-form payload with no built-in notion of sender
- Topics are only used for message routing and are not "created" or "destroyed"
- By default, messages are not stored, and a message with no client subscribing to its topic is simply discarded
- There are no specified way of querying connected clients or current subscriptions
- An MQTT client can use wildcards when subscribing to topics

Because of these properties, it is impossible to discover devices that do not publish any message.

Usually a device will publish into topics that are specific to that device instance so that it becomes possible to know the origin of the message.
For example a weather station might publish into topics like `weather/<city>/temperature` and `weather/<city>/pressure` with `<city>` the location of station.

Many power-constrained devices are not maintaining a long-lived connection to the MQTT broker.

In the rest of the document we assume that devices to discover follow this kind of pattern.

Using the above description of MQTT and device behavior, if we consider the Discovery Handler to be an MQTT client, we can define a discoverable device as an MQTT client that will "frequently" publish to an MQTT topic.

## Discovery Details and Properties

In order to discover MQTT based devices, we need the following pieces of information:

- The MQTT broker to connect to (and its credentials)
- The topic patterns to subscribe to
- How long shall we wait to consider a silent device as disconnected

Here is the expected schema for the discovery details:

```yaml
discoveryDetails:
    mqttBrokerUri:
        type: string
    topics:
        type: array
        items:
            type: string
    timeoutSeconds: 
        type: int64
        minimum: 0
        exclusiveMinimum: true
```

If the following keys are present in the discovery properties they will be used for connection and authentication to the MQTT broker:

- `mqtt_broker_username`: username for authentication against the broker
- `mqtt_broker_password`: password for authentication against the broker
- `mqtt_broker_certificate`: client TLS certificate for authentication against the broker
- `mqtt_broker_key`: client TLS key for authentication against the broker
- `mqtt_broker_ca`: TLS CA bundle for validating the broker TLS certificate (if unspecified, system-wide bundle is used)

## Instance properties

The ID of the device is the full topic used by the device.

The following properties shall be forwarded to the broker or any instance requesting the device:

- `MQTT_BROKER_URI`: the MQTT broker URI
- `MQTT_TOPIC`: The topic the message was received on

## Discovery process

The discovery handler must subscribe to all topics pattern specified in the `topics` array on the `mqttBrokerUri` MQTT broker.

The discovery handler must keep track of all discovered devices.

When receiving a message on a watched topic, the discovery handler must:

- Add the device to its discovered device list if not already here
- Reset the timer of the device to `timeoutSeconds`

When the timer of a device expires, the discovery handler must:

- Delete the device from its discovered device list

Every time the discovered device list changes, the discovery handler must send that list back to the Agent.
