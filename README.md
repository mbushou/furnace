Furnace: Self-Service Tenant VMI for the Cloud
=======

<img align="right" width="20%" src="https://github.com/mbushou/furnace/blob/master/misc/logo_smoke_sm.png">

Introduction
-------
Although Virtual Machine Introspection (VMI) tools are increasingly capable,
modern multi-tenant cloud providers are hesitant to expose the sensitive
hypervisor APIs necessary for tenants to use them.

This project, Furnace, is an open source VMI
framework that satisfies both a tenant's desire to run their own
custom VMI tools underneath their cloud VMs and a cloud
provider's expectation of security.

For additional details on Furnace's motivation and design, please
[check out](https://link.springer.com/chapter/10.1007/978-3-030-00470-5_30) the Furnace paper.

This repository contains information about the overall project.  The following
individual repositories contain the actual software components.

- [Furnace Sandbox](https://github.com/mbushou/furnace_sandbox)
- [Furnace Proxy](https://github.com/mbushou/furnace_proxy)
- [Furnace Backend](https://github.com/mbushou/furnace_backend)
- ~~[Furnace Cloud
    Service](https://github.com/mbushou/furnace_cloud_service)~~
    (under development!)
- ~~[Furnace Hypervisor
  Agent](https://github.com/mbushou/furnace_hypervisor_agent)~~ (under
  development!)
- ~~[Furnace Proxy
    Agent](https://github.com/mbushou/furnace_proxy_agent)~~ (under
    development!)

The individual repositories listed above are intended to be installed on specific cloud
infrastructure components (e.g., the Furnace sandbox is installed on each cloud
compute node).  The diagram below shows Furnace's overall software
architecture and which repository belongs on which cloud component.

<img align="center" width="100%" src="https://github.com/mbushou/furnace/blob/master/misc/cloud_management.png">

Warning!
-------
- Furnace is a young project that is very much a work in progress.
- Until Furnace is more mature, it is not recommended to be used in a production cloud.
- See an issue?  Report it!

Installation
-------
See [INSTALL.md](INSTALL.md) for instructions on installing Furnace in a single-hypervisor configuration.

Built with
-------
* [Bubblewrap](https://github.com/projectatomic/bubblewrap)
* [ZeroMQ](http://zeromq.org/)
* [Protocol Buffers](https://developers.google.com/protocol-buffers/)
* [Fedora](https://getfedora.org/)
* [Xen](https://www.xenproject.org/)
* [LibVMI](https://github.com/libvmi/libvmi)
* [DRAKVUF](https://github.com/tklengyel/drakvuf)

Citation
-------
Using Furnace for something?  Cite us!

```
@inproceedings{18RAID_Furnace,
 title = {{Furnace: Self-Service Tenant VMI for the Cloud}},
 author = {Bushouse, Micah and Reeves, Douglas},
 bookTitle={21st International Symposium on Research in Attacks, Intrusions, and Defenses},
 year = {2018},
 location = {Heraklion, Crete, Greece},
}
```

FAQ
-------

##### I have a problem with Furnace, how can I get help?

For general issues and issues with installation, please create an issue on this repo.  Issues related to a specific Furnace repo should be posted to that repo.

##### Do I have to use DRAKVUF?  What about Furnace on KVM?

The hypervisor-specific component of Furnace is its VMI partition.  Presently,
we recommend DRAKVUF with the Furnace plugin for this partition, however this
limits us to Xen hypervisors.

Furnace can be made to support any hypervisor that supports LibVMI.  A swap-in
replacement for DRAVKUF is under development, which would make Furance
compatible with KVM hypervisors.


##### What's with the logo?

In memory forensics, virtual machines (and hosts in general) are occasionally depicted as a collection of kernel and process address spaces.  These address spaces are represented as the "smoke" rising above the flames (virtual machine introspection actions).  A Furnace app is shown as a yellow shield at the center running underneath the VM.

License
-------
Furnace is GPLv3.

However, to use Furnace library with DRAKVUF, you must also comply with DRAKVUF's license.
Including DRAKVUF within commercial applications or appliances generally
requires the purchase of a commercial DRAKVUF license (see
https://github.com/tklengyel/drakvuf/blob/master/LICENSE).
