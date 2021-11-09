# Install Sandbox Containers in Openshift 4

## Install the Sandbox Container Operator:

* Check the version of the OpenShift cluster: 

oc version -o json | jq -r .openshiftVersion
4.9.5

* Clone the repository with the OCP objects to the Sandbox Containers:  

```sh
git clone https://github.com/rcarrata/sandbox-containers.git
cd sandbox-containers
```

* Install the subscription for the Sandbox Containers Preview 1.1:

```sh
oc apply -k operator/overlays/preview-1.1/
```

* Check the sandbox containers operator subscription in the sandbox namespace:

```sh
oc get subs -n openshift-sandboxed-containers-operator
NAME                            PACKAGE                         SOURCE             CHANNEL
sandboxed-containers-operator   sandboxed-containers-operator   redhat-operators   preview-1.1
```

* Check the ClusterServiceVersion of the Sandbox container Operator:

```sh
oc get csv -n openshift-sandboxed-containers-operator
NAME                                   DISPLAY                                   VERSION   REPLACES                               PHASE
sandboxed-containers-operator.v1.1.0   OpenShift sandboxed containers Operator   1.1.0     sandboxed-containers-operator.v1.0.2   Succeeded
```

* Check that the Operator is up and running:

```sh
oc get deployments -n openshift-sandboxed-containers-operatoror
NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
sandboxed-containers-operator-controller-manager   1/1     1            1           20s
```

NOTE: Notice that it's in the Phase Succeeded

## Creating and configuring the KataConfig in Openshift cluster

You must create one KataConfig custom resource (CR) to trigger the OpenShift sandboxed containers Operator to do the following:

1. Install the needed RHCOS extensions, such as QEMU and kata-containers, on your RHCOS node.
2. Ensure that the runtime, CRI-O, is configured with the correct Kata runtime handlers.
3. Create a RuntimeClass custom resource with necessary configurations for additional overhead caused by virtualization and the required additional processes.

* You can selectively install the Kata runtime on specific workers. Let's select one:

```
NODE_KATA=$(oc get node -l node-role.kubernetes.io/worker= --no-headers=true | awk '{ print $1 }' | head -n1)

echo $NODE_KATA
ocp-8vr6j-worker-0-82t6f
```

* Label the node that will be where the KataConfig will apply:

```
oc label node $NODE_KATA kata=true
```

* In the KataConfig we can see the matchLabel with the exact label defined in the step before:

```sh
apiVersion: kataconfiguration.openshift.io/v1
kind: KataConfig
metadata:
  name: cluster-kataconfig
 spec:
    kataConfigPoolSelector:
      matchLabels:
         kata: 'true'
```

Note: There is a typo in the doc [1] since it uses true as label value instead of ‘true’

* Install the KataConfig using kustomize:

```sh
oc apply -k instance/overlays/default/
```

* You can monitor the values of the KataConfig custom resource by running:

```sh
oc describe kataconfig cluster-kataconfig
Name:         cluster-kataconfig
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  kataconfiguration.openshift.io/v1
Kind:         KataConfig
...
Spec:
  Kata Config Pool Selector:
    Match Labels:
      Kata:  true
Status:
  Installation Status:
    Is In Progress:  True
    Completed:
    Failed:
    Inprogress:
  Prev Mcp Generation:  2
  Runtime Class:        kata
  Total Nodes Count:    1
  Un Installation Status:
    Completed:
    Failed:
    In Progress:
      Status:
  Upgrade Status:
Events:  <none>
```

as you can notice the KataConfig installation it's in progress in One Total Node Count and using the Runtime Class kata

You can check to see if the nodes in the machine-config-pool object are going through a config update.

```
oc get mcp kata-oc
NAME      CONFIG                                              UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
kata-oc   rendered-kata-oc-7ce66b5e9e1c51753ebf99c1d9603bd8   True      False      False      1              1                   1                     0                      3m
```

If we check the machine config pool we can check that the kata-oc mcp it's based in several machine configs, and in specific there is one that defines the sandbox-containers extensions:

```
oc get mcp kata-oc -o yaml | egrep -i 'kata|sandbox'
  name: kata-oc
    name: rendered-kata-oc-7ce66b5e9e1c51753ebf99c1d9603bd8
      name: 50-enable-sandboxed-containers-extension
      - kata-oc
      kata: "true"
    message: All nodes are updated with rendered-kata-oc-7ce66b5e9e1c51753ebf99c1d9603bd8
    name: rendered-kata-oc-7ce66b5e9e1c51753ebf99c1d9603bd8
      name: 50-enable-sandboxed-containers-extension
```

You can check the machine config that uses the mcp of kata-oc:

```sh
oc get mc | grep sand
50-enable-sandboxed-containers-extension                                                      3.2.0
4m
```

Let's check in detail this MachineConfig described in the step before:

```sh
oc get mc $(oc get mc | awk '{ print $1 }' | grep sand) -o yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
...
  labels:
    app: cluster-kataconfig
    machineconfiguration.openshift.io/role: worker
  name: 50-enable-sandboxed-containers-extension
...
spec:
  config:
    ignition:
      config:
        replace:
          verification: {}
      proxy: {}
      security:
        tls: {}
      timeouts: {}
      version: 3.2.0
    passwd: {}
    storage: {}
    systemd: {}
  extensions:
  - sandboxed-containers
  fips: false
  kernelArguments: null
  kernelType: ""
  osImageURL: ""
```

as you can noticed the extensions have the sandbox-containers. The [RHCOS extensions](https://github.com/openshift/machine-config-operator/blob/master/docs/MachineConfiguration.md#rhcos-extensions) users can enable a limited set of additional functionality on the RHCOS nodes. In 4.8+ only [usbguard and sandboxed-containers](https://docs.openshift.com/container-platform/4.9/post_installation_configuration/machine-configuration-tasks.html#rhcos-add-extensions_post-install-machine-configuration-tasks) are supported extensions

Automatically the Machine Config Operator will apply the MachineConfigPool rendered containing the different MachineConfigs, including the last MC of sandboxed-containers-extension described in the previous step.

```
oc get nodes -l kubernetes.io/os=linux,node-role.kubernetes.io/worker=
NAME                       STATUS                     ROLES    AGE   VERSION
ocp-8vr6j-worker-0-82t6f   Ready,SchedulingDisabled   worker   26h   v1.22.0-rc.0+a44d0f0
ocp-8vr6j-worker-0-kvxr9   Ready                      worker   26h   v1.22.0-rc.0+a44d0f0
ocp-8vr6j-worker-0-sl79n   Ready                      worker   26h   v1.22.0-rc.0+a44d0f0
```

As you can check the first selected worker labeled with the kata: 'true' is automatically installing and configuring the KataConfig extension.

We can check when the Machine Config Operator finishes their work, when the Updating, and Degraded sections of the MCP are in False state:

```sh
# oc get mcp kata-oc
NAME      CONFIG                                              UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
kata-oc   rendered-kata-oc-999617e684ef816391be460254498d18   True      False      False      1              1                   1                     0                      8m33s
```

## CRIO Kata Runtimes configurations

Now, let's check the configuration of the CRIO, if it's configured correctly for the Kata runtime handlers.

Let's jump to the worker node labeled with the Kata containers:

```sh
# oc debug node/$NODE_KATA
Starting pod/ocp-8vr6j-worker-0-82t6f-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.126.53
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host bash
[root@ocp-8vr6j-worker-0-82t6f /]#
```

If we check the crio.conf.d folder, we can see the 50-kata file containing the specifics of the Kata runtimes for CRIO:

```
[root@ocp-8vr6j-worker-0-82t6f /]# cat /etc/crio/crio.conf.d/50-kata
[crio.runtime.runtimes.kata]
  runtime_path = "/usr/bin/containerd-shim-kata-v2"
  runtime_type = "vm"
  runtime_root = "/run/vc"
  privileged_without_host_devices = true
```

## Analysis of the RuntimeClass / Kata Runtime

RuntimeClass defines a class of container runtime supported in the cluster. The RuntimeClass is used to determine which container runtime is used to run all containers in a pod.

RuntimeClasses are manually defined by a user or cluster provisioner, and referenced in the PodSpec. In our case we will have the RuntimeClass "kata", because we will demoing the Sandbox Containers use in our OpenShift clusters.

After a while, the Kata runtime is now installed on the cluster and ready for use as a secondary runtime. 

Let's verify that we have a newly created RuntimeClass for Kata on our cluster.

```sh
oc get runtimeClass kata -o yaml
apiVersion: node.k8s.io/v1
handler: kata
kind: RuntimeClass
metadata:
  name: kata
...
overhead:
  podFixed:
    cpu: 250m
    memory: 350Mi
scheduling:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
```

