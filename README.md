# Sandbox Containers in OpenShift 4

```
oc apply -k operator/overlays/preview-1.1/
```

```
oc get subs -n openshift-sandboxed-containers-operator
NAME                            PACKAGE                         SOURCE             CHANNEL
sandboxed-containers-operator   sandboxed-containers-operator   redhat-operators   preview-1.1
```

```
oc get csv -n openshift-sandboxed-containers-operator
NAME                                   DISPLAY                                   VERSION   REPLACES                               PHASE
sandboxed-containers-operator.v1.1.0   OpenShift sandboxed containers Operator   1.1.0     sandboxed-containers-operator.v1.0.2   Succeeded
```

```
apiVersion: kataconfiguration.openshift.io/v1
kind: KataConfig
metadata:
  name: cluster-kataconfig
```

```
oc apply -k instance/overlays/default/
```

```
oc get mc | grep sand
50-enable-sandboxed-containers-extension                                                      3.2.0
16m
```

```
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



## Links of Interest

* [Sandbox Continers 101](https://cloud.redhat.com/blog/openshift-sandboxed-containers-101)
