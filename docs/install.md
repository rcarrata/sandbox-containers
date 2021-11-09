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

## Triggering the installation of the Kata runtime

You must create one KataConfig custom resource (CR) to trigger the OpenShift sandboxed containers Operator to do the following:

1. Install the needed RHCOS extensions, such as QEMU and kata-containers, on your RHCOS node.
2. Ensure that the runtime, CRI-O, is configured with the correct Kata runtime handlers.
3. Create a RuntimeClass custom resource with necessary configurations for additional overhead caused by virtualization and the required additional processes.

* You can selectively install the Kata runtime on specific workers. Let's select one:

```
NODE_KATA=$(oc get node -l node-role.kubernetes.io/worker= --no-headers=true | awk '{ print $1 }' | head -n1)

echo $NODE_KATA
worker-0.sharedocp4upi49.lab.upshift.rdu2.redhat.com
```

```
oc label node $NODE_KATA kata=true
```

```sh
apiVersion: kataconfiguration.openshift.io/v1
kind: KataConfig
metadata:
  name: cluster-kataconfig
 spec:
    kataConfigPoolSelector:
      matchLabels:
         kata: true
```


```sh
oc apply -k instance/overlays/default/
```



```sh
oc get mc | grep sand
50-enable-sandboxed-containers-extension                                                      3.2.0
16m
```

```sh
oc get mc $(oc get mc | awk '{ print $1 }' | grep sand) -o yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  creationTimestamp: "2021-11-08T19:03:59Z"
  generation: 1
  labels:
    app: cluster-kataconfig
    machineconfiguration.openshift.io/role: worker
  name: 50-enable-sandboxed-containers-extension
  resourceVersion: "644597"
  uid: a7f33dfa-025a-4f7d-83c8-adf9d33823f1
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
