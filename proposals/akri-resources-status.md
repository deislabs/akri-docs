# Status of akri resources

Currently, both akri managed resources (`Instances` and `Configurations`) only have a `spec` and `metadata` field, this doesn't allow the akri
components to give feedback to the user. In order to do so, we can add a `status` field to the resources, this document describes how this field
should be populated.

## Instances

For an instance, the status should give insight about the broker resources (if any), to do that, you can use the following conditions:

- Healthy: The instance is currently detected by an agent, "True" (instance detected) and "Unknown" (in grace period) are the only valid statuses
- BrokerScheduled: The broker resources are all created (absent if no broker)
- BrokerReady: All broker resources are "Ready" or "Succeeded"
- Ready: All above conditions are True

When the BrokerScheduled or Ready condition is False, the message shall include what resources are the cause of this.

In order to be able to correctly set the "Healthy" condition, a list of healthy nodes (must be a subset of `spec.nodes` field) is needed.

Here is an example of the status of an Instance:

```yaml
status:
  conditions:
    - type: Healthy
      status: "True"
      lastTransitionTime: "2023-07-24T10:40:00Z"
      massage: ""
      reason: Discovered
    - type: BrokerScheduled
      status: "True"
      lastTransitionTime: "2023-07-24T10:40:00Z"
      message: ""
      reason: AllResourcesScheduled
    - type: BrokerReady
      status: "False"
      lastTransitionTime: "2023-07-24T10:40:00Z"
      message: "broker-pod is not Ready"
      reason: BrokerNotReady
    - type: Ready
      status: "False"
      lastTransitionTime: "2023-07-24T10:40:00Z"
      message: "broker-pod is not Ready"
      reason: BrokerNotReady
  healthyNodes:
    - node-a
    - node-b
```

## Configurations

For a configuration, the status should give insight about the Configuration itself, and its Instances:

- Started: At least an agent has the matching discovery handler, and the discovery handler is discovering this configuration
- InstancesReady: All instances are Ready (absent if no instances)
- Ready: all above conditions are True

Here is an example of the status of a Configuration:

```yaml
status:
  conditions:
    - type: Started
      status: "True"
      lastTransitionTime: "2023-07-24T10:40:00Z"
      massage: ""
      reason: DiscoveryHandlerStarted
    - type: InstancesReady
      status: "False"
      lastTransitionTime: "2023-07-24T10:40:00Z"
      massage: "Instance foo-bar-00095f is not Ready"
      reason: InstanceNotReady
    - type: Ready
      status: "False"
      lastTransitionTime: "2023-07-24T10:40:00Z"
      massage: "Instance foo-bar-00095f is not Ready"
      reason: InstanceNotReady
```
