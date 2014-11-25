==========================================
Compass Containerization
==========================================

https://blueprints.launchpad.net/compass/+spec/compass-containerization

Compass will use docker to contain its modules(core module, OS Installer 
and Package Installer) to make Compass more robust and flexible to setup
and tear down. This will solve many related problems as well, such as 
installation latencies in different regions, software/OS release version 
conflicts and so on.

Problem description
===================

Currently there is a set of installation scripts to setup an all-in-one
Compass. It has following prolbems:

* Operating-system dependant: Compass installation scripts can only run on
  CentOS-based OS now. Making Compass work on other Operating Systems would
  require a large portion of the installation code to be re-written.

* Prolonged installation time: Depending on region, bandwidth and latency,
  a successful Compass installation could take up to more than 90 minutes.

* Unstable downloading: Currently Compass is downloading many files from the
  internet during its installation(rpms, isos and some zip files). Some of 
  file sources are unstable, which causes installation failure from time to
  time.

* System "pollution": Once Compass gets installed on a server, it is almost
  impossible to tear down or wipe off.


Proposed change
===============

The proposal is to improve Compass installation by containerizing Compass 
core module, OS installer module and package installer module. Detailed
proposed changes are shown below:

**Containerization:**

Containerization of Compass and its modules should have three scenarios below
and all of them should be covered in this implementation:

1. Three containers exist in one host:

.. image:: ../images/all_dockers_in_1_host.png

- In this scenario, the host will contain three docker containers: Compass,
  Cobbler and Chef.
- Each container will have respective ports exposed and linked to certain 
  ports on the host.
- Log files for progress updating will be stored in Union File System, a 
  file sharing data volume provided by docker.
- Chef container's admin key pair will be injected during its setup, a copy
  of which will be stored in Compass for knife to work.
- Potential issue: There will be potential port management issues.

2. Three containers exist in three hosts:

.. image:: ../images/3container_in_3host.png

- In this scenario, three hosts are set up to contain Compass modules.
- OS installation log files will be stored in cobbler container(by cobbler's
  nature) and transported using rsyslog back to the Compass container.
- Chef's admin key pairs will be dealt with similarly to scenario #1.
- Potential issue: Communication between containers becomes more complicated.

3. Only Compass core modules are contained:

.. image:: ../images/1_compass_container.png

- In this scenario, only Compass is contained.
- Cobbler and Chef servers, whether in containers or hosts, preexist and are 
  reachable from Compass host and Compass container.
- Cobbler and Chef servers ssh credentials will be needed.

Above three different scenarios will be triggered based on user's preferences,
when running Compass installation. A more systematic view of this work flow
is described below.

**System Workflow:**

* Following changes should be made:

  * A new Github repository designated for Compass installation.

  * On master branches of compass-core/compass-adapters, stable CL's need to be
    tagged as "stable".

  * Above can be achieved by having a special periodic(weekly) check job that
    checks the lastest change and if it passes, it will be tagged.

  * An automatic job needs to be added to pull the newest stable CL, update the
    docker images and push them to the docker registry. This can be called as 
    the "image updatign" job.

|

* Given the above changes added to Compass, the brief workflow could be:

  * Initially, Compass, Cobbler and Chef docker images will be pushed to docker
    registry.

  * On every Friday at mid-night, the stable check job will be triggered to
    pull the latest merged CL and test it with a more thorough coverage and
    if it passes, tag that CL as "stable".

  * Once a CL is tagged as "stable", the "image updating" job will pull that CL,
    updating the existing image(the image from last successful build), and push
    it to the docker registry to replace the existing image.

  * Compass-adapters also should be checked periodically to have CL's tagged as
    "stable". The "image updating" job should also be triggered once adapters
    repo has newly "stable" tagged CLs.

  * Once user has cloned compass-install(?) repository and made choices of what
    the system should look like, the new installing scripts(playbooks) will
    pull docker images from docker registry and configure containers.

|

* A detailed example workflow is shown below:

.. image:: ../images/new_workflow.png

* Above example shows one possible scenario, numbers stand for time:

  1. At midnight on a Friday, both check job for compass-core and
     compass-adapters are triggered. It is assumed here that they start
     simutaneously.
  2. Check job for compass-adapters starts(same time as #1).
  3. compass-core check job starts running.
  4. compass-adapters check job starts running(same time as #3).
  5. adapters check job succeeds first, it tags the CL as "stable".
  6. After tagging "stable" to the CL, adapter check job triggers
     "image updating" job to update compass image from latest stable adapters
     CL.
  7. "image updating" job pulls newest stable CL from compass-adapters.
  8. "image updating" job starts running.
  9. Now compass-core check job succeeds and tags the CL as "stable".
  10. compass-core check job triggers "image updating". But there is an
      updating job running, it has to wait in queue(till #12).
  11. "image updating" job for compass-adapters finishes, replacing the image
      on docker registry.
  12. "image updating" job for compass-core starts.
  13. "image updating" job for compass-core finishes, replacing the image on
      docker registry again.
  14. On Monday, a user runs the installation script, it pulls the latest
      image from docker registry.

Alternatives
------------

* Using Ansible to directly pull compass-core and adapters and inject them
  into target hosts.

  * there is no "stable" concept.

  * docker containers are easier to set up/tear down.

* Run chef-solo on host

  * writing chef cookbooks definitely takes longer than building images from 
    stable CL's.

Data model impact
-----------------

New variables:
  * compass_server
  * cobbler_server
  * chef_server

Cobbler kickstart metadata field will be modified to accompany this change.
No database migrations will be required to accompany this change. 
Initially these variables will be generated by entry point of compass-install,
where users will input these values.

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

User interface for installing compass on command line will look different.
Instead of asking detailed questions, now compass will ask only for basic and
intuitive questions.

Performance Impact
------------------

Impacts:

* No major impact to overall performance

* Improves performance during compass installation.

* A few more check jobs need to be added to CompassCI.

Other deployer impact
---------------------

Added configuration options:

* Deploy cobbler/chef or not?

* If they exist, IPs?

Immediate impact after merging:
None
 
Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  xichengchang
  <xicheng.chang@huawei.com>

Work Items
----------

* Build docker images

* Re-write entry point of compass-install

* Create jobs for stable check

* Create jobs for image updating

* Write Dockerfiles for three images

* Write Ansible playbooks for configuration

* Configure rsyslog/nfs for log files syncing.

Dependencies
============

None

Testing
=======

Continuous Integration will be updated accordingly. If the current regtest
aren't to be changed, a specific job to check containerization should be created.

Documentation Impact
====================

Major impact on Compass production documents.

References
==========

None
