---
layout: post
title: "New Virt features in Juno"
date: 2014-09-12 12:37:17 +0800
comments: true
categories: 
---

There are three Virt features in this release, it all about cpu and memory. As the describe in the nova specs, they are used for NFV cases.

https://github.com/openstack/nova-specs/blob/master/specs/juno/virt-driver-cpu-pinning.rst
https://github.com/openstack/nova-specs/blob/master/specs/juno/virt-driver-large-pages.rst
https://github.com/openstack/nova-specs/blob/master/specs/juno/virt-driver-numa-placement.rst
https://github.com/openstack/nova-specs/blob/master/specs/juno/virt-driver-vcpu-topology.rst

All of four features is mark as implemented, but as I see, HugePages and CPUPinning aren't implemented all.

HugePages
=========

This feature aims to improve teh libvirt driver so that it can use huge pages as backing the guest RAM allocation. This will improve the performance of guest workloads by increasing TLB cache effciency. And it can ensure the guest has 100% dedicated RAM that will never be swapped out.

To implement this feature, it need virt driver support and schedule support. Virt driver need to know how to allocate huge page for guest. Scheduler need to know which compute node have enough HugePages for the new instances.

But I didn't saw those are implemented, I only find out a config class for memoryBacking: https://github.com/openstack/nova/blob/81b1babcd9699118f57d5055ff9045e275b536b5/nova/virt/libvirt/config.py#L1509, But there isn't anyone use it.

There are two blog post to introduce the HugePages from IBMer:
https://www.ibm.com/developerworks/community/blogs/fe313521-2e95-46f2-817d-44a4f27eba32/entry/backing_guests_with_hugepages?lang=en
https://www.ibm.com/developerworks/community/blogs/fe313521-2e95-46f2-817d-44a4f27eba32/entry/benefits_of_huge_pages?lang=en


CPU Pinning
===========

This feature aims to support the policy of pinning vcpu to pcpu. It want to support two type policy:

* hw:cpu_policy=shared|dedicated
* hw:cpu_threads_policy=avoid|separate|isolate|prefer

It need scheduler to know the compute node cpu pinning status. But all of this didn't implement yet.

And Numa Placement feature also need to depend on cpu pinning support to pin vcpu to a set of pcpu in one cells.

For now, there only support one config option vcpu_pin_set for now. That option specified which pcpu can be used for guest.


VCPU Topology
=============

This feature aims to support define the vcpu topology for guest. This enable to avoid some VCPU topology limitation for some OS vendors. For example, Windows didn't support 8 cpu as 8 sockets, 1 core.

* hw:cpu_sockets=NN - preferred number of sockets to expose to the guest
* hw:cpu_cores=NN - preferred number of cores to expose to the guest
* hw:cpu_threads=NN - preferred number of threads to expose to the guest
* hw:cpu_max_sockets=NN - maximum number of sockets to expose to the guest
* hw:cpu_max_cores=NN - maximum number of cores to expose to the guest
* hw:cpu_max_threads=NN - maximum number of threads to expose to the guest

We can use above option in flavor extra data and image properties to define the vcpu topology. The default vcpu topology for kvm is maxium sockets and 1 cores.
This feature just need implement in virt driver. Whatever the guest vcpu topology looks like it won't effect the instance scheduling.

In libvirt, the vcpu topology define as:

{% codeblock lang:xml %}
  <cpu>
    <topology sockets='8' cores='1' threads='1'/>
  </vcpu>
{% endcodeblock %}

Numa Placement
==============

This feature is landed at last minute before rc. This feature aims to enable inteligent NUMA node placement. It need virt driver support, scheduler support, and resource tracker support.

* Virt driver support pinning all VCPUs of a guest cell to all PCPUs of a physical cell. it also consider option vcpu_pin_set
* Resource tracker support collect the host numa topology, and the topology usage of guest, and report to scheduler.
* Scheduler support choice one host can fit guest numa topology. 1. The host must have more cells than guest. 2. And each cell have enough cpu and memory allocated to guest. (use ram_ratio and cpu_ratio)

In libvirt, the numa topology define as:

{% codeblock lang:xml %}
  <vcpu placement='static'>8</vcpu>
  <cputune>
    <vcpupin vcpu='0' cpuset='0-7,16-23'/>
    <vcpupin vcpu='1' cpuset='0-7,16-23'/>
    <vcpupin vcpu='2' cpuset='0-7,16-23'/>
    <vcpupin vcpu='3' cpuset='0-7,16-23'/>
    <vcpupin vcpu='4' cpuset='8-15,24-31'/>
    <vcpupin vcpu='5' cpuset='8-15,24-31'/>
    <vcpupin vcpu='6' cpuset='8-15,24-31'/>
    <vcpupin vcpu='7' cpuset='8-15,24-31'/>
  </cputune>
  <cpu>
    <topology sockets='8' cores='1' threads='1'/>
    <numa>
      <cell cpus='0-3' memory='1024'/>
      <cell cpus='4-7' memory='1024'/>
    </numa>
  </cpu>
{% endcodeblock %}


Some problems:

* The problem of current schedule, it hasn't good balance between cells in one host. For example, host have four cells, all the two cells guest will only pin to the frist two cells in host
* 1 socket, 8 threads also can be work with numa node.
* Resource tracker didn't track the threads for cpu. https://bugs.launchpad.net/nova/+bug/1355921

Others
======

* I/O (PCIe) based Numa scheduling: https://github.com/openstack/nova-specs/blob/master/specs/juno/input-output-based-numa-scheduling.rst
  This isn't landed in Juno. It aims to consider the host PCI device numa topology when scheduling instance. It target to Kilo now.

* Support virtio-scsi bus support for volume: https://github.com/openstack/nova-specs/blob/master/specs/juno/add-virtio-scsi-bus-for-bdm.rst
  hw_scsi_model property from volume's glance_image_metadata

* vhost-user support: https://github.com/openstack/nova-specs/blob/master/specs/juno/vif-vhostuser.rst
  This is used for userspace switch, like DPDK

* Ironic dirver support: https://github.com/openstack/nova-specs/blob/master/specs/juno/add-ironic-driver.rst

* Enable qemu memory ballon stats: https://github.com/openstack/nova-specs/blob/master/specs/juno/enabled-qemu-memballoon-stats.rst

* disk discard option: https://github.com/openstack/nova-specs/blob/master/specs/juno/libvirt-disk-discard-option.rst







