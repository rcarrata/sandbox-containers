# Sandbox Containers in OpenShift 4

OpenShift sandboxed containers support for OpenShift Container Platform provides users with built-in support for running Kata Containers as an additional optional runtime. This is particularly useful for users who are wanting to perform the following tasks:

* Run privileged or untrusted workloads.
* Ensure kernel isolation for each workload.
* Share the same workload across tenants.
* Ensure proper isolation and sandboxing for testing software.
* Ensure default resource containment through VM boundaries.

OpenShift sandboxed containers also provides users the ability to choose from the type of workload that they want to run to cover a wide variety of use cases.

You can use the OpenShift sandboxed containers Operator to perform tasks such as installation and removal, updates, and status monitoring.

NOTE: Sandboxed containers are only supported on bare metal.

## Guide for the PoC

* [Install Sandbox Containers](docs/install.md)

* [Sandbox Containers PoC Guide](docs/poc.md)


NOTE: This PoC it's tested in 4.9.5 using the Sandbox Containers Preview 1.1.
