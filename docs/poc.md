# Guide for PoC Sandbox Containers in OCP4

Let's deploy two identical pods, one using the regular RuntimeClass and the other using the RuntimeClass "kata" to check the differences between them.

Both pods will use the exact same image (net-tools) pulled from Quay.io.

## Deploying a regular workload

First, let's generate a new project:

```
oc new-project test-kata
```

Let's deploy a simple example using the net-tools image and a sleep command:

```
oc apply -f examples/example-net-regular.yaml
```

```
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

```
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

```
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

```sh
node_kata=$(oc get pod/example-fedora -o=jsonpath='{.spec.nodeName}')

echo $node_kata
worker-0.sharedocp4upi49.lab.upshift.rdu2.redhat.com
```

```sh
oc get pod/example-fedora -o=jsonpath='{.status.containerStatuses[*].containerID}' | cut -d "/" -f3
```



```sh
oc debug node/$node_kata
chroot /host bash
[root@worker-0 /]#
```

```sh
[root@worker-0 /]# ps aux | grep qemu
root       12325  0.6  4.0 2463548 332908 ?      Sl   19:10   0:15 /usr/libexec/qemu-kiwi -name
sandbox-43c05e1a05b14528d9a8173bfd23787ee9235635f512af9c6cb521b223368fda -uuid
872150e8-1fba-44f1-b445-966bc52d0a5d -machine q35,accel=kvm,kernel_irqchip
...
```

```sh
QEMU=$(crictl inspect 639f725e7724630af7345315bd678192eb45539f70b3ef2a01a457d689f3a2f | jq -r '.info.sandboxID')

echo $QEMU
43c05e1a05b14528d9a8173bfd23787ee9235635f512af9c6cb521b223368fda
```

```sh
[root@worker-0 /]# ps aux | grep qemu | grep $QEMU
root       12325  0.6  4.0 2463548 332908 ?      Sl   19:10   0:15 /usr/libexec/qemu-kiwi -name
sandbox-43c05e1a05b14528d9a8173bfd23787ee9235635f512af9c6cb521b223368fda

[root@worker-0 /]# echo $?
0
```

