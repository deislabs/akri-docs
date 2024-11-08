# Split Discovery related and Workload related Configuration elements

Currently, Akri's Configuration object contains both elements regarding the discovery of devices (`discoveryDetails`, `brokerProperties`, `discoveryProperties` and `capacity`), and elements regarding the "brokers" that are to be scheduled (`brokerSpec`, `instanceServiceSpec`, `configurationServiceSpec`).
This blurs the responsibility of the resources and make their management more complex.

This proposal divides the `Configuration` object in two that are described in the following sections.

## Note about the scope of objects

The `Configuration` and `Instance` objects are currently namespaced, this makes a lot of sense as the "brokers" will obviously be namespaced and as the `Instances` are linked to their `Configuration` they must share its namespace.
However, when splitting the `Configuration` in two, it makes sense to reconsider that choice as devices are not bound to a namespace per se. This proposal then makes the device description object cluster wide, as well as the `Instance` object.

## Note about naming

The objects, as well as our documentation, currently use the term "broker" to refer to the Pod or Job that get scheduled on every node for every discovered device.
This is ill-named as it can be much more than a mediator, in this proposal, when referring to the new objects, the term "workload" will be used.

## Description of the devices to discover: `DiscoveryConfiguration`

The `DiscoveryConfiguration` object is a cluster-wide object with all information needed to discover devices:

- The name of the discovery handler: `discoveryHandlerName`
- A string that a Discovery Handler knows how to parse to obtain necessary discovery details: `discoveryDetails`
- A set of extra properties the Discovery Handler may need, that can be pulled from `ConfigMaps` or `Secrets` at runtime: `discoveryProperties`
- The number of slots for a device instance: `capacity`
- A set of extra properties that will get added to the `Instance` properties and forwarded to workloads using the device: `extraInstancesProperties`

## Description of the kubernetes objects to create: `WorkloadConfiguration`

The `WorkloadConfiguration` object is a namespaced object that will contain the following properties:

- The name of the `DiscoveryConfiguration` whose `Instances` shall trigger the scheduling of the resources described in this `WorkloadConfiguration`: `discoverySelector`
- The spec part of the workload to schedule: `workloadSpec`
- The optional spec part of the `Service` to spawn pointing to all workload scheduled by this `WorkloadConfiguration`: `configurationServiceSpec`
- The optional spec part of a `Service` to spawn for every `Instance` that triggers this `WorkloadConfiguration`: `instanceServiceSpec`
