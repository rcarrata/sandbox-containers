# Guide for PoC Sandbox Containers in OCP4

Let's deploy two identical pods, one using the regular RuntimeClass and the other using the RuntimeClass "kata" to check the differences between them.

Both pods will use the exact same image (net-tools) pulled from Quay.io.

## Deploying a regular workload

First, let's generate a new project:

```sh
oc new-project test-kata
```

Let's deploy a simple example using the net-tools image and a sleep command:

```sh
oc apply -f examples/example-net-regular.yaml
```

```sh
apiVersion: v1
kind: Pod
metadata:
  name: example-net-tools-regular
spec:
  nodeSelector:
    kata: 'true'
  containers:
    - name: example-net-tools-regular
      image: quay.io/rcarrata/net-toolbox:latest
      command: ["/bin/bash", "-c", "sleep 99999999999"]
```

if you noticed, the pod have a nodeSelector with the label for ensure that the workloads are running in the labeled kata container worker.

```sh
oc get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE                       NOMINATED NODE   READINESS GATES
example-net-tools-regular   1/1     Running   0          55s   10.128.2.41   ocp-8vr6j-worker-0-82t6f   <none>           <none>
```

## Creating a Kata Container workload

* Let's deploy the same pod but with the runtimeClass defined as 'kata':

```sh
oc apply -f examples/example-net-kata.yaml
```

```sh
# cat examples/example-net-kata.yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-net-tools-kata
spec:
  nodeSelector:
    kata: 'true'
  containers:
    - name: example-net-tools-kata
      image: quay.io/rcarrata/net-toolbox:latest
      command: ["/bin/bash", "-c", "sleep 99999999999"]
```

Let's confirm that the pods are running with the runtimeClassName as Kata:

```sh
oc get pod example-net-tools-kata -o jsonpath='{.spec.runtimeClassName}'
kata

oc get pod example-net-tools-kata -o jsonpath='{.spec.runtimeClassName}' -o yaml | grep runtimeClass | tail -1
  runtimeClassName: kata
```

## Analysis of the Kata Containers Workloads and Qemu VMs

As all we know, workloads in Kubernetes/OpenShift runs within a Pod, the minimal representation of a workload in Kubernetes/OpenShift.

In case of the Kata Containers / Sandbox Containers, pods are bound to a QEMU Virtual Machine (qemu-vm), which provides this extra layer of isolation and furthermore extra layer of security.

Each VM runs in a qemu process and hosts a kata-agent process that acts as a supervisor for managing container workloads and processes that are running in those containers.

Let's see how this works comparing the regular pod and the pod using kata container RuntimeClass.

* First let's compare the uptime between the regular pod and the kata container pod:

A. Sandboxed Workloads

```sh
oc exec -ti example-net-tools-kata -- cat /proc/uptime
1082.84 1077.77
```

B. Regular Container

```sh
oc exec -ti example-net-tools-regular -- cat /proc/uptime
3074.51 11793.48
```

For what reason is there a difference so significant? The uptime of the standard container kernel is the same as the uptime of the node that is running the container, while the uptime of the sandboxed kata container pod is of its sandbox (VM) that was created at the pod creation time.

* On the other hand, let's compare the kernel command lines from the /proc/cmdline:

A. Sandboxed Workloads

```sh
oc exec -ti example-net-tools-kata -- cat /proc/cmdline
tsc=reliable no_timer_check rcupdate.rcu_expedited=1 i8042.direct=1 i8042.dumbkbd=1 i8042.nopnp=1 i8042.noaux=1 noreplace-smp reboot=k console=hvc0 console=hvc1 cryptomgr.notests net.ifnames=0 pci=lastbus=0 debug panic=1 nr_cpus=4 scsi_mod.scan=none agent.log=debug
```

B. Regular Container

```sh
oc exec -ti example-net-tools-regular -- cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt3)/ostree/rhcos-9d5687c13259b146f9e918ae7ee8e03f697789ddb69e37058072764a32d96fc4/vmlinuz-4.18.0-305.19.1.el8_4.x86_64 random.trust_cpu=on console=tty0 console=ttyS0,115200n8 ignition.platform.id=qemu ostree=/ostree/boot.0/rhcos/9d5687c13259b146f9e918ae7ee8e03f697789ddb69e37058072764a32d96fc4/0 root=UUID=7a2353f2-a105-4dfa-b800-49030ae7725e rw rootflags=prjquota
```

As we can see in the output of both cmdline commands, the containers are using two different kernel instances running in different environments (a worker node and a lightweight VM).

* On the third place, we will check the number of cores on both containers:

A. Sandboxed Workloads

```sh
oc exec -ti example-net-tools-kata -- cat /proc/cpuinfo | grep processor | wc -l
1
```

B. Regular Container

```sh
oc exec -ti example-net-tools-regular -- cat /proc/cpuinfo | grep processor | wc -l
4
```

By default, Sandbox Containers are configured to run with one vCPU per VM, and for this reason we're seeing the difference between the container running in Sandbox Containers and the regular containers which shows the total amount of vCPUs available in the worker node.

* Finally let's check the kernel used in each container:

A. Sandboxed Workloads

```sh
oc exec -ti example-net-tools-kata -- cat /proc/cpuinfo | grep processor | wc -l
1
```

B. Regular Container

```sh
oc exec -ti example-net-tools-kata -- uname -r
4.18.0-305.19.1.el8_4.x86_64
```

But they are the same kernel version! But we checked that are different kernel instances in the cmdline before, what's happening?

Despite of the mentioned differences, Kata Containers used by the OpenShift Sandbox Containers it's always running with the very same kernel version on the VM as the underlying RHCOS is running with (OS). The VM image is generated at the startup of the host, ensuring that is compatible with the kernel currently used by the RHCOS host.

## Analysis of the QEMU processes of the Sandbox Containers

Sandbox Containers are containers that are sandboxed by a VM running a QEMU process.

If we check the node where the workload is running, we can verify that this QEMU process is also running there:

* Let's check which node our example kata pod has been assigned to:  

```sh
pod_node_kata=$(oc get pod/example-net-tools-kata -o=jsonpath='{.spec.nodeName}')

echo $pod_node_kata
ocp-8vr6j-worker-0-82t6f
```

as we expected is the same as we defined using the label kata: 'true'.

* Let's extract the containerID CRI:

```sh
oc get pod/example-net-tools-kata -o=jsonpath='{.status.containerStatuses[*].containerID}' | cut -d "/" -f3
fcc17d03b182e0ac3db33f3a19668a7b59d1ef32934e0af0db01a6a482c04056
```

* Let's jump into the node where the kata container is running using the oc debug node:

```sh
oc debug node/$pod_node_kata
chroot /host bash
[root@worker-0 /]#
```

* Check the qemu processes that are running in the node:

```sh
ps aux | grep qemu
sandbox-712a4cb4a28dff8655bd92fd6bd5e761173a2b2c23b8b7615a5bbb12ca1b75a3 -uuid 2428acef-210f-42c5-9def-7c407c5c4042 -machine q35,accel=kvm,kernel_irqchip -cpu host,pmu=off
...
```

as we can see, there is one qemu process running. It is expected to see a qemu process running for each pod running sandbox containers / kata containers on that host/node.

* Let's check the sandboxID with the crictl inspect. We will use the containerID from the earlier step:

```sh
sandboxID=$(crictl inspect fcc17d03b182e0ac3db33f3a19668a7b59d1ef32934e0af0db01a6a482c04056 | jq -r '.info.sandboxID')

echo $sandboxID
712a4cb4a28dff8655bd92fd6bd5e761173a2b2c23b8b7615a5bbb12ca1b75a3
```

* Check the QEMU process again filtering by the sandboxID:

```sh
[root@worker-0 /]# ps aux | grep qemu | grep $sandboxID
root       12325  0.6  4.0 2463548 332908 ?      Sl   19:10   0:15 /usr/libexec/qemu-kiwi -name
sandbox-712a4cb4a28dff8655bd92fd6bd5e761173a2b2c23b8b7615a5bbb12ca1b75a3

[root@worker-0 /]# echo $?
0
```

The QEMU process is indeed running the container we inspected, because the CRI sandboxID is associated with your containerID. The Ids from the crictl inspect that outputs the sandboxID and the QEMU process running the workload are the same.
