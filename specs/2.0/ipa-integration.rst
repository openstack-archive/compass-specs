..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============
IPA Integration
===============

https://blueprints.launchpad.net/compass/+spec/ipa-integration

Ironic Python Agent provides a software defined(smart) way to program and
manage hardware. It can collect hardware specs, download and write images
to disk, perform RAID(?) and firmware upgrade, hardware topology discovery,
etc. It also provides a pluggable platform that could embed different
drivers for 'vendor proprietary utilities]<https://github.com/rackerlabs\
/onmetal-ironic-hardware-manager/blob/master/onmetal_ironic_hardware_manager\
/__init__.py>'
There are two ways to take advantage of IPA capabilities: 
1. Integrate with IPA directly, we write our own agent driver module and
ipmitool driver, move what ironic does and what not to compass or cobbler.
2. Integrate with Ironic. we don't have to write our own agent and ipmitool
driver, but Ironic should be able to be decoupled from keystone/glance/neutron
for image and dhcp service as standalone application.


Problem description
===================

There are a list of things that we want to accomplish or the current solution
hasnot resolve well yet.

* Firmware/Bios Upgrade: NA from current solution.
* RAID setup: NA from current solution.
* Hardware Discovery: Poll Switch. This approach depends on arp table on the
  switch learned from the arp broadcast packets sent by baremetals, which has
  aging issue causing possibly several retries or even reboots of the baremetal
  machines. Those actions are not even automatic and involves human intervention.
  Moreover, not all switches support snmp polling.
* Topology: Poll Switch. People have to know the switches' mgmt IP and snmp info.
* Spec Inspection: NA from current solution. Hence, no flaovr filtering for
  certain hardware dependent roles.
* OS Provisioning: By cobbler. Traditional pxe boot plus kickstarts. tftp and http
  are not easy to scale as the deployment size goes up. It is not efficient to
  download packages one by one. One time provisioning without programmability.


Proposed change
===============

Option 1: Integrate with a standalone Ironic, which is under development by the
following specs:
https://blueprints.launchpad.net/ironic/+spec/support-external-dhcp
https://blueprints.launchpad.net/ironic/+spec/non-glance-image-refs
Part of the flow can be smiliar with that of nova ironic driver.
#. Compass calls Ironic API to create bare metal nodes with images & driver info.

#. Compass calls Ironic API to create ports for the nodes.

#. Compass pushes mac addresses to the dhcp server's white list.

#. Compass calls Ironic API to set provision state to active to spawn nodes.

#. Compass waits for the nodes to finish deployment to be active.

#. For other advanced features provided by IPA and extension, Compass calls the
   vendor passthru interface.

* Pros:

#. Take advantage of the existing capabilities of Ironic, such as pxe and ipmi
   driver, heart beat interactions with IPA.

* Cons:

#. Decoupling of dhcp and image provider is stil not complete, having to wait
   or push for it to be ready. We will also have to prepare the images and
   metadata ourselves offline.

Option 2: Integrate with IPA directly.

#. Compass embeds pxe and ipmitool driver along with agent driver for power
   management, pxe boot to deploy agent on ramdisk.

#. Compass borrows the existing agent driver in ironic for talking to IPA on
   node validation, writing images to disk, heatbeat interaction.

#. Compass uses the passthru interface to employ other methods such as spec
   inspection(interface, cpu, memory, disk), erase disk, firmware upgrade
   bios configration and update, LLDP topology info, provided by IPA and its
   extensions, onmetal hardware manager for instance.

* Pros:

#. Have direct control over IPA, more flexibility. Does not need Ironic to be
   part of Compass, simplify the Compass structure.

* Cons:

#. On the other hand, loses the existing capalibilities of Ironic and have to
   reinvent the wheels or adapt those in Ironic into Compass.

* Follow-up question:

Q: How to propagate the initial configuration of chef-client/rsyslog/local-repo/
network config/... into the image provisioned by IPA?
A: Config drive plus metadata?

Alternatives
------------

#. Implement the whole OpenStack or Keystone/Nova?/Ironic/Neutron/Glance/Swift/Heat?
   stack to take advantage of the existing orchestration engine.
#. Take only Keystone/Ironic/Neutron/Glance/Swift? stack for OS provisioning part.

Data model impact
-----------------

#. Templates for Ironic and IPA APIs
#. Power Status/Provisioning Status which don't exist in Compass
#. Flavor based on hardware specs
#. Firmware/bios versioning
#. RAID

REST API impact
---------------

#. Power Management
#. Heartbeat interaction with IPA if Compass talks to IPA directly
#. specs/firmware/bios/raid/topo if Compass needs to terminate these info
   instead of passthru or passthru API alternatively.

UI Impact
---------

#. Topology view
#. DHCP/Image Server host or URL
#. Firmware/Bios/RAID configuration/upgrade
#. Role assginment based on hardware specs/flavor

Cookbook Impact
---------------

#. If we replace cobbler with Ironic, some initial configs by snippets
   would have to be moved to chef cookbooks.

Installation Impact
-------------------

#. Install Ironic as hardware and OS provisioning engine.
#. If Compass provides dhcp/image server, those need to be installed.
   IPA image needs to be build and uploaded.

Security impact
---------------

#. pxe image/disk image authentication check
#. ipmi credentials needs to be stored securely

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

#. If Compass talks to IPA directly, there would be performance impact as cluster
   size rises, just like Ironic conductor.
#. If Compass talks to Ironic, there could be load balancing among Ironic nodes.

Other deployer impact
---------------------

#. Deployer needs to prepare dhcp server and image server and get images uploaded
   before deployment.
#. Deployer needs to provide IPMI IP info and credentials.
#. Deployer needs to enable switch LLDP support if he/she wants topology view from
   Compass.
#. Deployer needs to build IPA image with extension driver and vendor
   proprietary tools if any.

Developer impact
----------------

None


Implementation
==============


Assignee(s)
-----------

Compass Team

Work Items
----------

Will update once we come to a concensus.

Dependencies
============

External dhcp and image server for Ironic blueprint.

Testing
=======

Same testing method as previous implementaion, but we also needs to employ the
ssh_pxe driver in Ironic to control libvirt VMs for pxe boot.

Documentation Impact
====================

#. New concepts in Compass, such as flavor, raid, bios/firmware needs to be
   reflected.
#. Installation instruction should aslo be updated accordingly.

References
==========

https://github.com/openstack/ironic/blob/master/ironic/drivers/modules/agent.py
https://github.com/openstack/ironic-python-agent/blob/master/ironic_python_agent/extensions/standby.py
https://github.com/openstack/nova/blob/master/nova/virt/ironic/driver.py
https://github.com/rackerlabs/onmetal-ironic-hardware-manager/blob/master/onmetal_ironic_hardware_manager
http://specs.openstack.org/openstack/ironic-specs/specs/juno/agent-driver.html
