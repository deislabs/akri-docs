# Dynamic Resource Allocation plugin mode

## Background

Under the kubernetes SIG node, the Dynamic Resource Allocation plugin mechanism ([KEP #3063](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/3063-dynamic-resource-allocation#summary))
aims to provide a new way to expose devices to kubernetes resources. While it doesn't aim to completely replace the Device Plugin system we currently
use, it does overlap a lot with it.

This new system is relevant to Akri as it would ease the management of slots usage and shared devices, for both instance level and configuration level resources.

As a quick reference, the DRA uses a set of kubernetes resources to work: A ResourceClass that register a new class of devices that might be allocatable
(with parameters for the driver) and a ResourceClaim that describe a request for allocation of a resource (with parameters for the driver).
These resources all ultimately refer to a driver, that is a two components system comprised of a resource controller and a resource kubelet plugin.
The controller will help the scheduler on node choice, and receive (de)allocation requests for the devices. The kubelet plugin will give the kubelet
all needed information to expose the device to the workload for a given ResourceClaim. The tracking of available resources is completely up to the driver.

In comparison, the current Akri model revolves around Configurations and Instances. A configuration both describe how to discover devices and what to
schedule when they are discovered (broker logic). An instance just represent a discovered instance and the list of nodes where it is available.

This proposal primarily focus on the Configuration resource, as the Instance resource is all about available resource tracking that is explicitly not
covered by DRA.

The Configuration resource match with the ResourceClass for the discovery part, the ResourceClaim part is handled by a combination of the Instance and
inner workings of the akri agent.

## API Changes

These API changes may require writing a migration tool for existing Configurations

### Create a `BrokerTemplate` resource kind

In order to fit in the DRA model we need to separate the broker logic from the discovery logic, thus we need to create a new resource kind to handle the broker part. This resource needs to be linked to a discovery logic resource and provides all the needed information about what to schedule.

This uses the "Deploy arbitrary Kubernetes resources as broker" for the description of what to schedule, please see that proposal for further details
on the exact content of the broker fields.

```yaml
# apiVersion got ommited on purpose
kind: BrokerTemplate
metadata: ...
spec:
    discoveryReference: foo
    instanceBrokerResources: ...
    globalBrokerResources: ... # this field is named configurationBrokerResources in other proposal
```

### Create a `DiscoveryConfiguration` resource kind

This resource bears the discovery part of the current Configuration resource. It would be used as ResourceClass parameter.
You can note that we do **not** include the name of the discovery handler as it would be in the driver name in ResourceClass
as a subdomain of `driver.akri.sh` like e.g. `udev.driver.akri.sh`

```yaml
# apiVersion got ommited on purpose
kind: DiscoveryConfiguration
metadata: ...
spec:
    capacity: 1
    details:
        udevRules:
            - KERNEL=="video[0-9]*"
    properties: ... # discoveryProperties in current Configuration
```

### Create a `PropertyFilter` resource kind

This is a brand-new resource kind, it aims to be used as parameter to a ResourceClaim, it allows to further filter out
on the instances discovered from a ResourceClass when doing a claim. This is what allows to request a specific instance, rather
than using the "configuration level" claim.

The resource allows filtering on any property exposed by the discovery handler (including the resource identifier).

```yaml
# apiVersion got ommited on purpose
kind: PropertyFilter
metadata: ...
spec:
    filters: # All filters must match for the instance to be considered
        - key: foo
          value: bar
          mode: exclude
```

## Architecture and Behavior Changes

### Add a New Controller: the driverController

The DRA mechanism needs a driver specific resource controller that will do the allocate/de-allocate work as well as the node selection. As we split
the Configuration resource in two, we must create a new entity to handler this.

This controller watches over the ResourceClass to trigger a discovery with the given DiscoveryConfiguration, it also watches over ResourceClaims to
inform the scheduler about node selection and manage (de)allocation.

### Rename current controller to brokerController

As we split the broker management from the discovery management, and we have another controller, controller becomes too vague for naming the entity,
hence we rename it to brokerController.

The brokerController does not need to have write permissions to the Instances.

### Agent behavior changes

The agent becomes lighter, it no longer watches over Configurations. The agent registers a `ResourcePlugin` per connected discovery handler, upon
allocation request it generates the [CDI](https://github.com/cncf-tags/container-device-interface) device and gives it to the kubelet.

The communication with the DH can be changed to allow making use of CDI features (like hooks), the agent still create/updates the Instances
when discovering devices (and when a device disappears)

## Timeline and changes order

The DRA is not yet in beta phase, we should provide this new behavior along with the "old" one as long as all our supported kubernetes
versions don't have the needed features, however some parts of the new behavior can be implemented without waiting for DRA to be broadly available.

Here is a proposed timeline:

1. Rename current controller to brokerController (can be done without any prerequisite on DRA)
2. Make API changes under `akri.sh/v1`, implement Agent's new behavior and driverController for this API version (keeping old behavior for `akri.sh/v0`), this can be done as soon as DRA is available.
3. Deprecate Device Plugin mode (and `akri.sh/v0` API) when all our supported versions have DRA
4. Remove `akri.sh/v0` API

The DH interface can be augmented with CDI features anytime after step 2.

## Links for reference

- [KEP-3063: Dynamic resource allocation](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/3063-dynamic-resource-allocation/README.md)
- [Device Plugins 2.0: How to Build a Driver for Dynamic Resource Allocation - K Klues & Alexey Fomenko (KubeCon Europe 2023)](https://www.youtube.com/watch?v=_fi9asserLE)
- [Container Device Interface Specification](https://github.com/cncf-tags/container-device-interface/blob/main/SPEC.md)
