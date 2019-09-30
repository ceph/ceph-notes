QOS Orientation
===============

Goal
----

Add support for serving low throughput/high priority clients at low latency
will maintaining high total throughput.

A secondary goal is for this framework is to also solve the problem of
maintaining high background recovery and scrub throughput without disturbing
throughput and latency for clients when client IO is available.

Work So Far
-----------

MClock/DMClock
~~~~~~~~~~~~~~

mclock and dmclock [1] are algorithms for allocating IO resources among a set
of clients with different io allocations/priorities.  [2] is the current ceph
implementation of dmclock.  [3] is an old PR adding the client level hooks
and messages for propagating the dmclock parameters. [4] is a cleaned pending
PR for a cleaned up reimplementation of the server side.

- [1] https://www.usenix.org/legacy/events/osdi10/tech/full_papers/Gulati.pdf
- [2] https://github.com/ceph/dmclock
- [3] https://github.com/ceph/ceph/pull/20235
- [4] https://github.com/ceph/ceph/pull/30650

In Progress:

- [4] above replaces the existing OpQueue based mclock implementations with a
  simpler one.

Next:

- Evaluate mclock behavior in scheduling client IO vs recovery/background IO
  (See Recovery below)

Throttles
~~~~~~~~~

A general prerequisite for providing QoS among competing sources of work is to
avoid accepting more work at any particular time than is required for the next
layer down to achieve the desired throughput.  The first and most important
place where this shows up in rados is at the bluestore layer.  Bluestore
currently will accept many IOs into the :bluestore queue before it begins to
block.  This means that if the next layer up receives a high priority IO, it
cannot be served until all requests already in the bluestore queue (presumably
lower priority) have been served.

In order to evaluate what the bluestore throttles *should* be set to, there is
now a set of tracepoints in bluestore [4] for measuring sampled per-io throttle
values and latency as well, modifications to the objectstore fio backend to
vary the throttle values over the course of a run, and a set of scripts in cbt
[5] for plotting the throttle/latency/throughput curves of the emitted
tracepoints.

Complete:

- [4] https://github.com/ceph/ceph/pull/29674

  * This PR is also a good primer on generally how to add lttng tracepoints.

- [5] https://github.com/ceph/cbt/pull/190

  * See fio_objectstore_tools/bluestore_throttle_tuning.rst in the cbt pr for
    more details on how to use these new tools.

Next Steps:

- Figure out how to automatically tune the throttles as the above tools aren’t reasonable for most users to use.

Recovery (future work)
~~~~~~~~~~~~~~~~~~~~~~

It’s possible that even without mclock, the existing WeightedPriorityQueue
implementation may suffice for controlling client IO vs recovery IO given
appropriate throttle values.

Next Steps:

- Add a way of measuring recovery operation throughput and latency, likely
  using lttng tracepoints as in [4] Use cbt’s existing recovery support to
  quantify recovery impact with and without appropriate throttle values.

