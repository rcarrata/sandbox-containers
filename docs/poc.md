## Guide for PoC Sandbox Containers in OCP4

* **Sandbox**: A sandbox is an isolated environment where programs can run. In a sandbox, you can run untested or untrusted programs without risking harm to the host machine or the operating system.

* **Pod in Sandbox Containers**: In the context of OpenShift sandboxed containers, a pod is implemented as a virtual machine. Several containers can run in the same pod on the same virtual machine.

* **OpenShift Sandbox Containers Operator**: The OpenShift sandboxed containers Operator is tasked with managing the lifecycle of sandboxed containers on a cluster.

* **Kata Containers**: Kata Containers is a core upstream project that is used to build OpenShift sandboxed containers. OpenShift sandboxed containers integrate Kata Containers with OpenShift Container Platform.

* **KataConfig**: KataConfig objects represent configurations of sandboxed containers. They store information about the state of the cluster, such as the nodes on which the software is deployed.

* **Runtime class**: A RuntimeClass object describes which runtime can be used to run a given workload. A runtime class that is named kata is installed and deployed by the OpenShift sandboxed containers Operator. 

```
oc apply -k examples/example-fedora-regular.yaml
```

```
oc apply -k examples/example-fedora-kata.yaml
```

```
oc get pod example-fedora -o jsonpath='{.spec.runtimeClassName}'
kata

oc get pod example-fedora -o yaml | grep runtimeClass | tail -1
  runtimeClassName: kata
```

```
oc exec example-fedora-regular -- cat /proc/uptime
1236.21 4595.80
```

```
oc exec example-fedora -- cat /proc/uptime
1274.88 1266.63
```

```
oc exec example-fedora-regular -- cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt3)/ostree/rhcos-a10b07df1aa66c008cd3b9acb17d765f0755702cadfa0090155dced4d2e9bfe0/vmlinuz-4.18.0-305.19.1.el8_4.x86_64 random.trust_cpu=on console=tty0 console=ttyS0,115200n8 ostree=/ostree/boot.0/rhcos/a10b07df1aa66c008cd3b9acb17d765f0755702cadfa0090155dced4d2e9bfe0/0 ignition.platform.id=openstack root=UUID=6756d05c-00bd-4616-8449-1eb2a83d56ed rw rootflags=prjquota
```

```
oc exec example-fedora -- cat /proc/cmdline
tsc=reliable no_timer_check rcupdate.rcu_expedited=1 i8042.direct=1 i8042.dumbkbd=1 i8042.nopnp=1 i8042.noaux=1 noreplace-smp reboot=k console=hvc0 console=hvc1 cryptomgr.notests net.ifnames=0 pci=lastbus=0 debug panic=1 nr_cpus=4 scsi_mod.scan=none agent.log=debug
```

```
node_kata=$(oc get pod/example-fedora -o=jsonpath='{.spec.nodeName}')

echo $node_kata
worker-0.sharedocp4upi49.lab.upshift.rdu2.redhat.com
```

```
oc get pod/example-fedora -o=jsonpath='{.status.containerStatuses[*].containerID}' | cut -d "/" -f3
```



```
oc debug node/$node_kata
chroot /host bash
[root@worker-0 /]#
```

```
[root@worker-0 /]# ps aux | grep qemu
root       12325  0.6  4.0 2463548 332908 ?      Sl   19:10   0:15 /usr/libexec/qemu-kiwi -name
sandbox-43c05e1a05b14528d9a8173bfd23787ee9235635f512af9c6cb521b223368fda -uuid
872150e8-1fba-44f1-b445-966bc52d0a5d -machine q35,accel=kvm,kernel_irqchip
...
```

```
QEMU=$(crictl inspect 639f725e7724630af7345315bd678192eb45539f70b3ef2a01a457d689f3a2f | jq -r '.info.sandboxID')

echo $QEMU
43c05e1a05b14528d9a8173bfd23787ee9235635f512af9c6cb521b223368fda
```

```
[root@worker-0 /]# ps aux | grep qemu | grep $QEMU
root       12325  0.6  4.0 2463548 332908 ?      Sl   19:10   0:15 /usr/libexec/qemu-kiwi -name
sandbox-43c05e1a05b14528d9a8173bfd23787ee9235635f512af9c6cb521b223368fda

[root@worker-0 /]# echo $?
0
```

