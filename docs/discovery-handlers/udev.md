# udev

## Background

Udev is the device manager for Linux. It manages device nodes in the `/dev` directory, such as microphones, security chips, usb cameras, and so on. Udev can be used to find devices that are attached to or embedded in Linux nodes.

All of Akri's components can be deployed by specifying values in its Helm chart during an installation. This document will cover the values that should be set to (1) deploy the udev Discovery Handlers and (2) apply a Configuration that tells Akri to discover devices using that Discovery Handler.

## Deploying the udev Discovery Handler

In order for the Agent to discover udev devices, a udev Discovery Handler must exist. Akri supports an Agent image that includes all supported Discovery Handlers. This Agent will be used if `agent.full=true` is set. By default, a slim Agent without any embedded Discovery Handlers is deployed and the required Discovery Handlers can be deployed as DaemonSets. This documentation will use that strategy, deploying udev Discovery Handlers by specifying `udev.discovery.enabled=true` when installing Akri.

## udev Configuration Settings

Instead of having to assemble your own udev Configuration yaml, we have provided a [Helm template](https://github.com/project-akri/akri/blob/main/deployment/helm/templates/udev-configuration.yaml). Helm allows us to parametrize the commonly modified fields in our configuration files, and we have provided many for udev (to see them, run `helm inspect values akri-helm-charts/akri`). To apply the udev Configuration to your cluster, simply set `udev.configuration.enabled=true` when installing Akri. Be sure to also **specify one or more udev rules** for the Configuration, as explained [below](#discovery-handler-discovery-details-settings).

### Discovery Handler Discovery Details Settings

Discovery Handlers are passed discovery details that are set in a Configuration to determine what to discover, filter out of discovery, and so on. The udev Discovery Handler requires that one discovery detail to be provided: [udev rules](https://wiki.archlinux.org/index.php/Udev).

| Helm Key | Value | Default | Description |
| :--- | :--- | :--- | :--- |
| udev.configuration.discoveryDetails.udevRules | array of udev rules | empty | udev rule [supported by the udev Discovery Handler](#udev-rule-format) |
| udev.configuration.discoveryDetails.groupRecursive | boolean | false | If set to true, group devices with a matching parent |
| udev.configuration.discoveryDetails.permissions | string | rwm | udev device mount permissions (**r**ead, **w**rite, **m**odify) |

The udev Discovery Handler parses the udev rules listed in a Configuration, searches for them using udev, and returns a list of discovered device nodes (ie: /dev/video0). It parses the udev rules via a grammar [grammar](https://github.com/project-akri/akri/blob/main/discovery-handlers/udev/src/udev_rule_grammar.pest) Akri has created. It expects the udev rules to be formatted according to the [Linux Man pages](https://linux.die.net/man/7/udev).
Additionally, the permissions for the device mount are defined as well and are set to `rwm` (read, write, modify) by default.

#### Udev rule format

While udev rules are normally used to both find devices and perform actions on devices, the Akri udev discovery handler is only interested in finding devices. Consequently, the discovery handler will throw an error if any of the rules contain an action operation ("=" , "+=" , "-=" , ":=") or action fields such as `IMPORT` in the udev rules. You should only use match operations ("==", "!=") and the following udev fields: `ATTRIBUTE`, `ATTRIBUTE`, `DEVPATH`, `DRIVER`, `DRIVERS`, `KERNEL`, `KERNELS`, `ENV`, `SUBSYSTEM`, `SUBSYSTEMS`, `TAG`, and `TAGS`. To see some examples, reference our example [supported rules](https://github.com/project-akri/akri/blob/main/test/example.rules) and [unsupported rules](https://github.com/project-akri/akri/blob/main/test/example-unsupported.rules) that we run some tests against.

### Broker Pod Settings

If you would like non-terminating workloads ("broker" Pods) to be deployed automatically to discovered cameras, a broker image should be specified (under `brokerPod`) in the Configuration. Alternatively, if it meets your scenario, you could use the Akri frame server broker ("ghcr.io/project-akri/akri/udev-video-broker"). If you would rather manually deploy pods to utilize the cameras advertized by Akri, don't specify a broker pod and see our documentation on [requesting resources advertized by Akri](../user-guide/requesting-akri-resources.md).

> Note only a `brokerJob` OR `brokerPod` should be specified.

| Helm Key | Value | Default | Description |
| :--- | :--- | :--- | :--- |
| udev.configuration.brokerPod.image.repository | image string | "" | image of broker Pod that should be deployed to discovered devices |
| udev.configuration.brokerPod.image.tag | tag string | "latest" | image tag of broker Pod that should be deployed to discovered devices |
| udev.configuration.brokerPod.resources.memoryRequest | string | "10Mi" | the minimum amount of RAM that must be available to this Pod for it to be scheduled by the Kubernetes Scheduler. Default based on the Akri udev sample broker. Adjust to the size of your broker. |
| udev.configuration.brokerPod.resources.cpuRequest | string | "10m" | the minimum amount of CPU that must be available to this Pod for it to be scheduled by the Kubernetes Scheduler. Default based on the Akri udev sample broker. Adjust to the size of your broker. |
| udev.configuration.brokerPod.resources.memoryLimit | string | "30Mi" | the maximum amount of RAM this Pod can consume. Default based on the Akri udev sample broker. Adjust to the size of your broker. |
| udev.configuration.brokerPod.resources.cpuLimit | string | "29m" | the maximum amount of CPU this Pod can consume. Default based on the Akri udev sample broker. Adjust to the size of your broker. |

### Broker Job Settings
If you would like terminating [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) to be deployed automatically to discovered devices, a broker image should be specified (under `brokerJob`) in the Configuration. A Kubernetes Job deploys a set number of terminating Pods.

> Note only a `brokerJob` OR `brokerPod` should be specified.

| Helm Key | Value | Default | Description |
| :--- | :--- | :--- | :--- |
| udev.configuration.brokerJob.image.repository | image string | "" | image of broker Job that should be deployed to discovered devices |
| udev.configuration.brokerJob.image.tag | tag string | "latest" | image tag of broker Job that should be deployed to discovered devices |
| udev.configuration.brokerJob.resources.memoryRequest | string | "10Mi" | the minimum amount of RAM that must be available to this Pod for it to be scheduled by the Kubernetes Scheduler. Default based on the Akri udev sample broker. Adjust to the size of your broker. |
| udev.configuration.brokerJob.resources.cpuRequest | string | "10m" | the minimum amount of CPU that must be available to this Pod for it to be scheduled by the Kubernetes Scheduler. Default based on the Akri udev sample broker. Adjust to the size of your broker. |
| udev.configuration.brokerJob.resources.memoryLimit | string | "30Mi" | the maximum amount of RAM this Pod can consume. Default based on the Akri udev sample broker. Adjust to the size of your broker. |
| udev.configuration.brokerJob.resources.cpuLimit | string | "29m" | the maximum amount of CPU this Pod can consume. Default based on the Akri udev sample broker. Adjust to the size of your broker. |
| udev.configuration.brokerJob.command | string array | Empty | command to be executed in the Pod |
| udev.configuration.brokerJob.restartPolicy | string array | `OnFailure` | `RestartPolicy` for the Job. Can either be `OnFailure` or `Never` for Jobs.|
| udev.configuration.brokerJob.backoffLimit | number | 2 | defines the Kubernetes Job [backoff failure policy](https://kubernetes.io/docs/concepts/workloads/controllers/job/#pod-backoff-failure-policy) |
| udev.configuration.brokerJob.parallelism | number | 1 | defines the Kubernetes Job [`parallelism`](https://kubernetes.io/docs/concepts/workloads/controllers/job/#parallel-jobs) |
| udev.configuration.brokerJob.completions | number | 1 | defines the Kubernetes Job [`completions`](https://kubernetes.io/docs/concepts/workloads/controllers/job) |

### Disabling Automatic Service Creation

By default, if a broker Pod is specified, the generic udev Configuration will create services for all the brokers of a specific Akri Instance and all the brokers of an Akri Configuration. The creation of these services can be disabled.

| Helm Key | Value | Default | Description |
| :--- | :--- | :--- | :--- |
| udev.configuration.createInstanceServices | true, false | true | a service should be automatically created for each broker Pod |
| udev.configuration.createConfigurationService | true, false | true | a single service should be created for all brokers of a Configuration |

### Capacity Setting

By default, if a broker Pod is specified, a single broker Pod is deployed to each device. To modify the Configuration so that a device is accessed by more or fewer nodes via broker Pods, update the `udev.configuration.capacity` setting to reflect the correct number. For example, if your high availability needs are met by having 1 redundant pod, you can update the Configuration like this by setting `udev.configuration.capacity=2`.

| Helm Key | Value | Default | Description |
| :--- | :--- | :--- | :--- |
| udev.configuration.capacity | number | 1 | maximum number of brokers that can be deployed to utilize a device (up to 1 per Node) |

## Choosing a udev rule

To see what devices will be discovered on a specific node by a udev rule, you can use `udevadm`. For example, to find all devices in the sound subsystem, you could run:

```bash
udevadm trigger --verbose --dry-run --type=devices --subsystem-match=sound
```

To see all the properties of a specific device discovered, you can use `udevadm info`:

```bash
udevadm info --attribute-walk --path=$(udevadm info --query=path /sys/devices/pci0000:00/0000:00:1f.3/sound/card0)
```

Now, you can see a bunch of attributes you could use to narrow your udev rule. Maybe you decide you want to find all sound devices made by the vendor `Great Vendor`. You set the following udev rule under the udev Discovery Handler in your Configuration:

```yaml
discoveryHandler:
  name: udev
  discoveryDetails: |+
    udevRules:
    -  'SUBSYSTEM=="sound", ATTR{vendor}=="Great Vendor"'
```

### Testing a udev rule

To test which devices Akri will discover with a udev rule, you can run the rule locally adding a tag action to it. Then you can search for all devices with that tag, which will be the ones discovered by Akri. 

1. Create a new rules file called `90-akri.rules` in the `/etc/udev/rules.d` directory, and add your udev rule(s) to it. For this example, we will be testing the rule `SUBSYSTEM=="sound", KERNEL=="card[0-9]*"`. Add `TAG+="akri_tag"` to the end of each rule. Note how 90 is the prefix to the file name. This makes sure these rules are run after the others in the default `70-snap.core.rules`, preventing them from being overwritten. Feel free to explore `70-snap.core.rules` to see numerous examples of udev rules.

```bash
      sudo echo 'SUBSYSTEM=="sound", KERNEL=="card[0-9]*", TAG+="akri_tag"' | sudo tee -a /etc/udev/rules.d/90-akri.rules
```

1. Reload the udev rules and trigger them.

   ```bash
    sudo udevadm control --reload
    sudo udevadm trigger
   ```

1. List the devices that have been tagged, which Akri will discover. Akri will only discover devices with device nodes (devices within the `/dev` directory). These device node paths will be mounted into broker Pods so the brokers can utilize the devices.

   ```bash
    udevadm trigger --verbose --dry-run --type=devices --tag-match=akri_tag | xargs -l bash -c 'if [ -e $0/dev ]; then echo $0/dev; fi'
   ```

1. Explore the attributes of each device in order to decide how to refine your udev rule.

   ```bash
    udevadm trigger --verbose --dry-run --type=devices --tag-match=akri_tag | xargs -l bash -c 'if [ -e $0/dev ]; then echo $0; fi' | xargs -l bash -c 'udevadm info --path=$0 --attribute-walk' | less
   ```

1. Modify the rule as needed, being sure to reload and trigger the rules each time.
1. Remove the tag from the devices -- note how  `+=` turns to `-=` -- and reload and trigger the udev rules. Alternatively, if you are trying to discover devices with fields that Akri does not yet support, such as `ATTRS`, you could leave the tag and add it to the rule in your Configuration with `TAG=="akri_tag"`.

   ```bash
      sudo echo 'SUBSYSTEM=="sound", KERNEL=="card[0-9]*", TAG-="akri_tag"' | sudo tee -a /etc/udev/rules.d/90-akri.rules
      sudo udevadm control --reload
      sudo udevadm trigger
   ```

1. Confirm that the tag has been removed and no devices are listed.

   ```bash
    udevadm trigger --verbose --dry-run --type=devices --tag-match=akri_tag
   ```

1. Create an Akri Configuration with your udev rule!

## Installing Akri with a udev Configuration and Discovery Handler

Leveraging the above settings, Akri can be installed with the udev Discovery Handler and a udev Configuration with our udev rule specified.

```bash
helm repo add akri-helm-charts https://project-akri.github.io/akri/
helm install akri akri-helm-charts/akri \
    --set udev.discovery.enabled=true \
    --set udev.configuration.enabled=true \
    --set udev.configuration.discoveryDetails.udevRules[0]='SUBSYSTEM=="sound"\, ATTR{vendor}=="Great Vendor"'
```

The following installation examples have been given to show how to the udev Configuration can be tailored to you cluster:

* Modifying the udev rule
* Specifying a broker pod image

For more advanced Configuration changes that are not aided by our Helm chart, we suggest creating a Configuration file using Helm and then manually modifying it. To do this, see our documentation on [Customizing an Akri Installation](../user-guide/customizing-an-akri-installation.md#generating-modifying-and-applying-a-custom-configuration)

## Modifying the udev rule

The udev Discovery Handler will find all devices that are described by ANY of the udev rules. For example, to discover devices made by either Great Vendor or Awesome Vendor, you could add a second udev rule.

```bash
helm repo add akri-helm-charts https://project-akri.github.io/akri/
helm install akri akri-helm-charts/akri \
    --set udev.discovery.enabled=true \
    --set udev.configuration.enabled=true \
    --set udev.configuration.discoveryDetails.udevRules[0]='SUBSYSTEM=="sound"\, ATTR{vendor}=="Great Vendor"' \
    --set udev.configuration.discoveryDetails.udevRules[1]='SUBSYSTEM=="sound"\, ATTR{vendor}=="Awesome Vendor"'
```

Akri will now discover these devices and advertize them to the cluster as resources. Each discovered device is represented as an Akri Instance. To list them, run `kubectl get akrii`. Note `akrii` is a short name for Akri Instance. All the instances will be named in the format `<configuration-name>-<hash>`. You could change the name of the Configuration and resultant Instances to be `sound-device` by adding `--set udev.configuration.name=sound-devices` to your installation command. Now, you can schedule pods that request these Instances as resources, as explained in the [requesting akri resources document](../user-guide/requesting-akri-resources.md).

## Specifying a broker pod image

Instead of manually deploying Pods to resources advertized by Akri, you can add a broker image to the udev Configuration. Then, a broker will automatically be deployed to each discovered device. The controller will inject the information the broker needs to find its device as environment variables. Namely, it injects an environment variable named `UDEV_DEVPATH_{INSTANCE_HASH}` which contains the device's sysfs path (i.e. `/devices/pci0000:00/0000:00:1f.3/sound/card0/input4`). Additionally, if the devnode path is found, it also injects an environment variable named `UDEV_DEVNODE_{INSTANCE_HASH}` which contains the devnode path for that device (i.e. `/dev/snd/pcmC0D0c`). The broker can grab these environment variables and proceed to interact with the device. To add a broker to the udev configuration, set the `udev.configuration.brokerPod.image.repository` value to point to your image. As an example, the installation below will deploy an empty nginx pod for each instance. Instead, you can point to your image, say `ghcr.io/<USERNAME>/sound-broker`.

```bash
helm repo add akri-helm-charts https://project-akri.github.io/akri/
helm install akri akri-helm-charts/akri \
    --set udev.discovery.enabled=true \
    --set udev.configuration.enabled=true \
    --set udev.configuration.discoveryDetails.udevRules[0]='SUBSYSTEM=="sound"\, ATTR{vendor}=="Great Vendor"' \
    --set udev.configuration.brokerPod.image.repository=nginx
```

> Note: set `udev.configuration.brokerPod.image.tag` to specify an image tag (defaults to `latest`).

Akri will automatically create a broker for each discovered device. It will also create a service for each broker and one for all brokers of the Configuration that applications can point to. See the [Customizing Akri Installation](../user-guide/customizing-an-akri-installation.md) to learn how to [modify the broker pod spec](../user-guide/customizing-an-akri-installation.md#modifying-the-brokerpodspec) and [service specs](../user-guide/customizing-an-akri-installation.md#modifying-instanceservicespec-or-configurationservicespec) in the Configuration.

### Setting the broker Pod security context

By default in the generic udev Configuration, the udev broker is run in privileged security context. This container [security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) can be customized via Helm. For example, to instead run all processes in the Pod with user ID 1000 and group 1000, do the following:

```bash
helm repo add akri-helm-charts https://project-akri.github.io/akri/
helm install akri akri-helm-charts/akri \
    --set udev.discovery.enabled=true \
    --set udev.configuration.enabled=true \
    --set udev.configuration.discoveryDetails.udevRules[0]='SUBSYSTEM=="sound"\, ATTR{vendor}=="Great Vendor"' \
    --set udev.configuration.brokerPod.image.repository=nginx \
    --set udev.configuration.brokerPod.securityContext.runAsUser=1000 \
    --set udev.configuration.brokerPod.securityContext.runAsGroup=1000
```

## Modifying a Configuration

Akri has provided further documentation on [modifying the broker PodSpec](../user-guide/customizing-an-akri-installation.md#modifying-the-brokerpodspec), [instanceServiceSpec, or configurationServiceSpec](../user-guide/customizing-an-akri-installation.md#modifying-instanceservicespec-or-configurationservicespec) More information about how to modify an installed Configuration, add additional Configurations to a cluster, or delete a Configuration can be found in the [Customizing an Akri Installation document](../user-guide/customizing-an-akri-installation.md).

## Grouping related device nodes

Akri currently provides a way to group device nodes under the topmost matching node, this allows to handle a complex device with multiple device nodes as one Instance.

For example with the following udev device tree and the rule `ENV{ID_SERIAL}=="Great Vendor Complex Camera"`:
```
root
├── P: /devices/root/device1
│   A: vendor=Great Vendor
│   E: ID_SERIAL=Great Vendor Complex Camera
│   ├── P: /devices/root/device1/video4linux/video0
│   │   A: vendor=Great Vendor
│   │   E: ID_SERIAL=Great Vendor Complex Camera
│   │   E: DEVNAME=/dev/video0
│   ├── P: /devices/root/device1/video4linux/video1
│   │   A: vendor=Great Vendor
│   │   E: ID_SERIAL=Great Vendor Complex Camera
│   │   E: DEVNAME=/dev/video1
│   └── P: /devices/root/device1/sound/card0/pcmC0D0c
│       A: vendor=Great Vendor
│       E: ID_SERIAL=Great Vendor Complex Camera
│       E: DEVNAME=/dev/snd/pcmC0D0c
└── P: /devices/root/device2
    A: vendor=Another Vendor
```
This would result in a single instance grouping `video0`, `video1` and `pcmC0D0c`.

All the device nodes will get mounted into the broker pod and will be listed in the environment variables with `UDEV_DEVNODE` prefix

This behavior can be enabled by setting the `udev.configuration.discoveryDetails.groupRecursive` to `true`.

## Implementation details

The udev implementation can be understood by looking at several things: 

1. [UdevDiscoveryDetails](https://github.com/project-akri/akri/blob/main/discovery-handlers/udev/src/discovery_handler.rs) defines the required properties 
1. [UdevDiscoveryHandler](https://github.com/project-akri/akri/blob/main/discovery-handlers/udev/src/discovery_handler.rs) defines udev discovery 
1. [samples/brokers/udev-video-broker](https://github.com/project-akri/akri/blob/main/samples/brokers/udev-video-broker) defines the udev broker 
1. [udev\_rule\_grammar.pest](https://github.com/project-akri/akri/blob/main/discovery-handlers/udev/src/udev_rule_grammar.pest) defines the grammar for parsing udev rules and enumerate which fields are supported (such as `ATTR` and `TAG`), which are yet to be supported (`ATTRS` and `TAGS`), and which fields will never be supported, mainly due to be assignment rather than matching fields (such as `ACTION` and `GOTO`).

