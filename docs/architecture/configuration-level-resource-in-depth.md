# Configuration-level Resources

Akri supports creating a Kubernetes resource (i.e. device plugin) for each individual device. Since each device in Akri is represented as an Instance custom resource, these are called Instance-level resources. Instance-level resources are named in the format `<configuration-name>-<instance-id>`. Akri also creates a Kubernetes Device Plugin for a Configuration called Configuration-level resource. A Configuration-level resource is a resource that represents all of the devices discovered via a Configuration. With Configuration-level resources, instead of needing to know the specific Instances to request, resources could be requested by the Configuration name and the Agent will do the work of selecting which Instances to reserve. The example below shows a deployment that requests the resource at Configuration level and would deploy a nginx broker to each discovered device respectively.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: onvif-camera-broker-deployment
  labels:
    app: onvif-camera-broker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: onvif-camera-broker
  template:
    metadata:
      labels:
        app: onvif-camera-broker
    spec:
      containers:
      - name: onvif-camera-broker
        image: nginx
        resources:
          limits:
            akri.sh/onvif-camera: "2"
          requests:
            akri.sh/onvif-camera: "2"
```

With Configuration-level resources, users could use higher level Kubernetes objects (Deployments, ReplicaSets, DaemonSets, etc.) or develop their own deployment strategies, rather than relying on the Akri Controller to deploy Pods to discovered devices.


### Maintaining Device Usage

The [in-depth resource sharing doc](resource-sharing-in-depth.md) describes how the `Configuration.capacity` and `Instance.deviceUsage` are used to achieve resource sharing between nodes.
The same data is used to achieve sharing the same resource between Configuration-level and Instance-level resources.

The `Instance.deviceUsage` in Akri Instances is extended to support Configuration device plugin.
The `Instance.deviceUsage` may look like this:

```yaml
  deviceUsage:
    my-resource-00095f-0: ""
    my-resource-00095f-1: ""
    my-resource-00095f-2: ""
    my-resource-00095f-3: "node-a"
    my-resource-00095f-4: ""
```
where empty string means the slot is free and non-empty string indicates the slot is used (by the node). To support Configuration device plugin, 
the `Instance.deviceUsage` format is extended to hold the additional information, the deviceUsage can be a "<node_name>" (for Instance) or a "C:<virtual_device_id>:<node_name>" (for 
Configuration). For example, the `Instance.deviceUsage` shows the slot `my-resource-00095f-2` is used by virtual device id "0" of the 
Configuration device plugin on `node-b`.  The slot `my-resource-00095f-3` is used by Instance device plugin on `node-a`.  The other 3 slots are 
free.

```yaml
  deviceUsage:
    my-resource-00095f-0: ""
    my-resource-00095f-1: ""
    my-resource-00095f-2: "C:0:node-b"
    my-resource-00095f-3: "node-a"
    my-resource-00095f-4: ""
```

### Configuration Device Plugin dynamic virtual device ids

The Instance device plugins report available resources to kubelet using fixed virtual device id. Instance device plugin construct the virtual device id by 
appending an index to the Instance name. For example, for an Instance device plugin, if the instance name is `akri.sh/akri-onvif-8120fe` and 
the `capacity` is 2, Instance device plugin reports 2 virtual devices `akri-onvif-8120fe-0` and `akri-onvif-8120fe-1`. This works fine for 
Instance device plugin but is not flexible for Configuration device plugin as we want to minimize the possibility that kubelet issues `allocate` 
requests that request more than available cameras. Instead of using fixed virtual device id, Configuration device plugins expose available 
resources using dynamic virtual device ids to provide maximum device usage flexibility.

Here is an example that two cameras are discovered for a Configuration (`akri.sh/akri-onvif-8120fe` and `akri.sh/akri-onvif-a19705`) and the 
`capacity` is 2.  The Configuration device plugin calculates available resources and, instead of reporting fixed id like `akri-onvif-8120fe-0`, 
the Configuration device plugin reports virtual device id "0", "1", ... as 'place holder' in `list_and_watch`. The actual device slot to be used 
is determined when `allocate` is called.

To avoid kubelet issues `allocate` requests that requests 3 cameras when only 2 cameras exist with a capacity of 2 each. The Configuration 
device plugin only report resources already being claimed by the Configuration device plugin and +1 for each camera that still has at least one 
free slot.  For our example, if all slots from 2 cameras are free, the Configuration device plugin reports 2 virtual devices "0" and "1" even 
there are actually 4 slots available to use.  In a different case, if `akri-onvif-8120fe-1` had been claimed by the Configuration device plugin 
as virtual device id "4" and all other 3 slots are free, the Configuration device plugin reports virtual device id "0", "1", and "4" as 
available virtual device ids.  The virtual device id "4" maps to `akri-onvif-8120fe-1` and id "0" and "1" can be mapped to 
`akri-onvif-8120fe-0`, `akri-onvif-a19705-0`, or `akri-onvif-a19705-1` later when `allocate` is called. By managing the number of available 
virtual devices, we can reduce the chances that kubelet issues `allocate` requests more than available cameras exist.

Note that there is still chances that all slots have been claimed by Configuration device plugin. For example, "0": `akri-onvif-8120fe-0`, "1": 
`akri-onvif-8120fe-1`, "2":`akri-onvif-a19705-0`, "3":`akri-onvif-a19705-1`.  In this case, Configuration device plugin reports all 4 virtual 
devices are available, and it's possible that kubelet requests "0" and "1" which map to the same camera.  In that case, the Configuration device 
plugin denies the allocation request and the Pod will be in an `UnexpectedAdmissionError` state.  The Configuration device plugin calculates 
virtual device availability periodically and reduces the number of available virtual devices when the reconciler detects slots not in-use and 
set the slot usage to free.  In our example, assume "3":`akri-onvif-a19705-1` is the only slot being used. The Agent reconciler sets slots 
`akri-onvif-8120fe-0`, `akri-onvif-8120fe-1`, and `akri-onvif-a19705-0` to free.
The Configuration device plugin then reduces the device availability to "0", "1" and "3".
If kubelet retries to claim "0" and "1", the Configuration device plugin will allow it by mapping "0" to `akri-onvif-8120fe-0` or `akri-onvif-8120fe-1`, "1" to `akri-onvif-a19705-0`.

The Configuration device plugin reports "0", "1", ... as virtual device ids in `list_and_watch` and determines the actual device slot to be used 
when `allocate` is called. The algorithm to map virtual device ids to actual device slot works on the allocation requests on a per-container 
basis that: 
- For a given container request, ensure allocated devices are unique instances
- If a virtual device id was claimed before, use the previous allocation information.
- If a virtual device id has being claimed, pick a device slot from the instance that has the most free slots.

For example, assume "3":`akri-onvif-a19705-1` is the only slot being used and other slots are all free. Configuration device plugin 
`list_and_watch` reports virtual device id "0", "1" and "3" are available (1 free from `akri-onvif-8120fe`, 1 free from `akri-onvif-a19705` and 
1 previously claimed from `akri-onvif-a19705`). 
- If kubelet requests "0" and "3" in a container request of an `allocate` call, the mapping is "0": `akri-onvif-8120fe-0` or 
`akri-onvif-8120fe-1` (since `akri-onvif-8120fe` has the most free slots between instance `akri-onvif-8120fe` (2) and `akri-onvif-a19705` (1))
and "3":`akri-onvif-a19705-1` (previously claimed).
- If kubelet requests "0", "1" and "3" in a container request of an `allocate` call, Configuration device plugin denies the request as there is 
only 2 cameras available to allocate.
- If kubelet request "0" and "1" in a container request and "3" in a different container request of an `allocate` call, the Configuratin device 
plugin maps "0": `akri-onvif-8120fe-0` or `akri-onvif-8120fe-1` (the most free slots) and "1":`akri-onvif-a19705-0` (unique instance) to 
container request 1 and "3":`akri-onvif-a19705-1` (previously claimed) to container request 2.


## Deployment Strategies with Configuration-level resources

The Akri Agent and Discovery Handlers enable device discovery and Kubernetes resource creation: they discover devices, create Kubernetes resources to represent
the devices, and ensure only `capacity` containers are using a device at once via the device plugin framework. The Akri Controller eases device use.
If a broker is specified in a Configuration, the Controller will automatically deploy Kubernetes Pods or Jobs to discovered devices. Currently the Controller only
supports two deployment strategies: either deploying a non-terminating Pod (that Akri calls a "broker") to each Node that can see a device or deploying a single
Job to the cluster for each device discovered. There are plenty of scenarios that do not fit these two strategies such as a ReplicaSet like deployment of n number
of Pods to the cluster. With Configuration-level resources, users could easily achieve their own scenarios without the Akri Controller, as selecting resources is
more declarative. A user specifies in a resource request how many OPC UA servers are needed rather than needing to delineate the exact ones already discovered by
Akri, as explained in Akri's current documentation on [requesting Akri resources](../user-guide/requesting-akri-resources.md).

For example, with Configuration-level resources, the following Deployment could be applied to a cluster:

```yaml
apiVersion: "apps/v1"
kind: Deployment
metadata:
  name: onvif-broker-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      name: onvif-broker
  template:
    metadata:
      labels:
        name: onvif-broker
    spec:
      containers:
      - name: nginx
        image: "nginx:latest"
        resources:
          requests:
            "akri.sh/akri-onvif": "2"
          limits:
            "akri.sh/akri-onvif": "2"
```


Pods will only be successfully scheduled to a Node and run if the resources exist and are available. In the case of the
above scenario, if there were two cameras on the network, two Pods would be deployed to the cluster. If there are not 
enough resources, say there is only one camera on the network,
the two Pods will be left in a `Pending` state until another is discovered. This is the case with any deployment on
Kubernetes where there are not enough resources. However, `Pending` Pods do not use up cluster resources.
