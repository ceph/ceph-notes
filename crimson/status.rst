Goal
====

Crimson is designed to be a faster OSD, in the sense that

- It targets fast storage devices, like NVMe storage, to take advantage of the high performance of random I/O and high throughput of the new hardware.
- It will be able to bypass the kernel when talking to networking/storage devices which support polling.
- It will be more computationally efficient, so the load is more balanced in a multi-core system, and each core will be able to serve the same amount of IO with lower CPU usage.
- It will achieve these goals by a design from scratch. It also utilizes modern techniques, like SPDK, DPDK and the Seastar framework, so it can 1) bypass the kernel, 2) avoid memcpy between, 3) avoid lock contention.

Crismon will be a drop-in-replacement of the classic OSD. Our long-term goal is to redesign the object storage backend from scratch, This new backend is dubbed “Seastore”. The disk layout we will use for Seastore won’t necessarily be optimal for HDDs. But Seastore is still in its very early stage, and is still missing a detailed design -- we don’t have enough resources to work on it yet. As an intermediate solution, we are working on adapting bluestore to crimson-osd in the hope that we can use it for testing f he mid-term. But if the new object storage backend does not work well with HDD, we will continue using bluestore in crimson just for supporting HDD.

We are using a C++ framework named Seastar to implement Crimson, but we are not porting classic OSD to Seastar, as the design philosophy and various considerations could be very different between them. So, again, please bear in mind, to port classic OSD to Seastar and adapt it to bluestore is never our final goal.

The new Crimson OSD will be compatible with classic OSD, so existing users will be able to upgrade from classic OSD to Crimson OSD. Put in other words, crimson-osd will:

- Support librados protocol
  * Be able to talk to existing librados clients
  * Be able to join an existing Ceph cluster as an OSD
- Support most existing OSD commands

Note that some existing options won’t apply to the Crimson OSD, and new options will be added.

Current status
==============

Currently, we are using memory-based Object Store for testing.

Functions
---------

The feature set of OSD can be covered by different use cases.

- rados benchmark

  * random/seq read/write
- RBD engine of the fio benchmark

  * random/seq read/write without exclusive locking (work in progress). This is
    enough to run the fio benchmark.

Tests
-----

Currently, we test crimson-osd in following ways:

- Unit test for different building blocks, like config, messenger, mon client.
  These tests are now part of “make check” run.
- Performance test for comparing the performance of a new change against that
  of master branch. This test is driven by CBT, and is now integrated as a
  jenkins job, which can be launched using a trigger phrase on GitHub.

Future Work
===========

all operations supported librados including snap support
--------------------------------------------------------
Which means we will be able to support RBD and RGW.

Log based recovery + backfill
-----------------------------

A core feature of ceph is online movement and recovery of data due to cluster
changes and failures. Broadly, the following features need to be added to
crimson to support those higher level features:

* Operation logging on write
* Recovery from the above for briefly down osds (Recovery)
* Online reconstruction for new osds/osds down for a longer period (Backfill)

Scrub
-----

Online verification of stored data over a long period of time is another staple
of the current osd -- this feature will need to be carried over to crimson.

I/O operation state machine
---------------------------

How the client I/O operations coordinate with background operations? For
instance, read/write I/O should wait for recovery/scrub/backfill etc.

Use config-proxy in messenger
-----------------------------

We are using a poor-man solution - a plain struct for holding msgr related
settings. And it’s not updated with the option subcomponent.

Performance Review and CI integration
-------------------------------------

Currently we have a preliminary CI support for checking of the significant
performance regressions, but we need to update the test harness and test cases
whenever it’s necessary. Also we need to

- Review the performance to see if there is any regression not caught by the CI
  tests.
- Compare the performance of classic OSD and crimson-osd on monthly basis
- Adapt crimson-osd to existing teuthology qa suite, once it is able to cover
  more rados operations, and hence support RBD and RGW clients.

Adapt to bluestore
------------------

We are now using a variant of memstore as the object store backend. Bluestore
will serve as the intermediate solution before we have seastore.

Seastore
--------

The next generation objectstore optimized for NVMe devices. And it would take
a long time before its GA. As a reference, bluestore started in early 2015, and
it was ready for production in late 2017. There will be three phases:

- Prototyping/Design:

  * Prototyping

    - define typical use cases, the constraints and assumptions.
    - evaluate different options and understand the tradeoffs, probably do some
      experiments, before solidifying on a specific design
  * Design:

    - in-memory/on-disk data structures
    - A sequence diagram to illustrate how to coordinate the foreground IOPS and
      background maintenance task.
    - define its interfaces talking to the other part of OSD
- Implementation:

  * Integrate the object store with crimson-osd. If seastore cannot support HDD
    well, it should be able to coexist with bluestore.
  * Stabilize the disk layout

Seastar+SPDK evaluation/integration
-----------------------------------

- To evaluate different approaches of kernel-bypassing techniques.
- To integrate SPDK into Seastar
