# Deploy arbitrary Kubernetes resources as broker

## Background

Currently, a broker is defined as either a Job or a Pod, and can be accompanied by a Configuration level Service and an Instance level service, as shown in following YAML example.

```yaml
apiVersion: akri.sh/v0
kind: Configuration
metadata:
  name: akri-debug-echo-foo
spec:
  brokerSpec:
    brokerPodSpec:
      containers:
      - name: akri-debug-echo-foo-broker
        image: "nginx:stable-alpine"
        resources:
          requests:
            "{{PLACEHOLDER}}" : "1"
            memory: 10Mi
            cpu: 10m
          limits:
            "{{PLACEHOLDER}}" : "1"
            memory: 30Mi
            cpu: 29m
  instanceServiceSpec:
    type: ClusterIP
    ports:
    - name: akri-debug-echo-foo-instance-service
      port: 6052
      protocol: TCP
      targetPort: 6052
  configurationServiceSpec:
    type: ClusterIP
    ports:
    - name: akri-debug-echo-foo-configuration-service
      port: 6052
      protocol: TCP
      targetPort: 6052
```

This limits the interoperability of Akri with other Kubernetes based components that would benefit from Akri creating CRs or other Kubernetes native resources.
For example Akri could act as the discovery component of a larger system that leverage Digital Twins with [Shifu](https://shifu.dev/).

## New broker definition

The idea behind the new broker definition is to provide two fields `instanceBrokerResources` and `configurationBrokerResources` that will contain a list of Kubernetes resources to deploy.

Instance level resources will get deployed for every instance, and Configuration level ones will get deployed once for a Configuration with at least a discovered Instance.

Akri will still add Configuration and Instance labels to resources it creates.

## New template variables

All those possibilities requires something a bit more advanced than the current `{{PLACEHOLDER}}` replacement, as the resources might need additional fields like the Instance name, or some of its properties.
To cover this let's have a couple of other "variables" that would replace the `PLACEHOLDER` one:

| Variable | Description |
| -------- | ----------- |
| INSTANCE_NAME | The name of the instance |
| INSTANCE_PROPERTIES | Dictionary with the `brokerProperties` of the Instance |
| LABEL_SELECTOR | Label Selector that matches instance level resources (only this instance when used in an instance level resource, all instances when used in configuration level resource)|
| NODES | List of node the instance is available on |

## Example

The previous example would the looks like this:

```yaml
apiVersion: akri.sh/v0
kind: Configuration
metadata:
  name: akri-debug-echo-foo
spec:
  instanceBrokerResources:
  - apiVersion: v1
    kind: Pod
    metadata:
      name: "{{INSTANCE_NAME}}-pod"
    spec:
      containers:
      - name: akri-debug-echo-foo-broker
        image: "nginx:stable-alpine"
        resources:
          requests:
            "akri.sh/{{INSTANCE_NAME}}" : "1"
            memory: 10Mi
            cpu: 10m
          limits:
            "akri.sh/{{INSTANCE_NAME}}" : "1"
            memory: 30Mi
            cpu: 29m
  - apiVersion: v1
    kind: Service
    metadata:
      name: "{{INSTANCE_NAME}}-svc"
    spec:
      type: ClusterIP
      ports:
      - name: akri-debug-echo-foo-instance-service
        port: 6052
        protocol: TCP
        targetPort: 6052
      selector: {{LABEL_SELECTOR}}
  configurationBrokerResources:
  - apiVersion: v1
    kind: Service
    metadata:
      name: akri-debug-echo-foo-svc
    spec:
      type: ClusterIP
      ports:
      - name: akri-debug-echo-foo-configuration-service
        port: 6052
        protocol: TCP
        targetPort: 6052
      selector: {{LABEL_SELECTOR}}
```

## Technical implementation ideas

This proposal implies many changes, and most importantly makes a breaking change to the CRD. It also implies a lot of changes to the controller and the admission webhook.

For the CRD itself, we can make use of `x-kubernetes-embedded-resource` that should help with validating the asked object is valid.

For the deployment, the RBAC would need to be much more permissive for the controller, probably allowing pretty much everything.

The admission webhook will also have to deal with mostly unspecified objects, one way to effectively solve this would be to try dry-run submit the resources to be created with a dummy instance.

The controller will have to watch over every object it owns without prior knowledge of its type.
