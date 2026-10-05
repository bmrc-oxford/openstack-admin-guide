.. include:: vars.rst

==============================
Ironic Node Burn-in
==============================

Workflows to onboard new hardware often include a stress-testing step to
provoke early failures, so that load-triggered issues do not only occur
once the node has already moved to production. These `burn-in` tests
typically cover the CPU, memory, disk and network (and GPUs where
present).

Ironic supports such tests as part of its cleaning framework: each test
runs as a cleaning step inside the Ironic Python Agent (IPA) ramdisk,
using standard tools:

- `stress-ng <https://wiki.ubuntu.com/Kernel/Reference/stress-ng>`__ for CPU and memory
- `fio <https://fio.readthedocs.io/en/latest/>`__ for disk and network
- `gpu-burn <https://github.com/wilicc/gpu-burn>`__ for GPU tests

Because the burn-in steps are part of the generic hardware manager in the
IPA, they are available without bundling a specific IPA hardware manager
into the ramdisk.

.. note::

   The |project_name| IPA image is built from source with the ``burn-in``
   DIB element (see ``ipa_build_dib_elements_extra`` in
   ``etc/kayobe/ipa.yml`` in the |kayobe_config| repository), so the burn-in
   tools are already present in the agent ramdisk. No IPA image rebuild is
   required before running the tests.

Running a burn-in test
----------------------

Burn-in steps are launched with ``baremetal node clean`` using an explicit
``--clean-steps`` argument that overrides the default cleaning steps. Each
test is configured with driver-info options on the node, all prefixed with
``agent_burnin_``.

Before starting, the node must be in the ``manageable`` or ``available``
provision state (see :ref:`ironic-node-lifecycle`). Run the commands below
as an administrator with OpenStack admin credentials loaded (see
:doc:`introduction` for the prompt conventions).

The examples use ``cloudcomp100`` as the node under test; substitute the
hostname or UUID of the node being burned in.

CPU burn-in
^^^^^^^^^^^

Available options (following the ``agent_burnin_`` + stress-ng stressor
(``cpu``) + stress-ng option schema):

- ``agent_burnin_cpu_timeout`` (default: 24 hours)
- ``agent_burnin_cpu_cpu`` (default: 0, meaning all CPUs)

to limit the overall runtime and to pick the number of CPUs to stress.

For instance, to limit the CPU burn-in on ``cloudcomp100`` to 10 minutes:

.. code-block:: console

   admin# openstack baremetal node set \
       --driver-info agent_burnin_cpu_timeout=600 cloudcomp100

Then launch the test:

.. code-block:: console

   admin# openstack baremetal node clean \
       --clean-steps '[{"step": "burnin_cpu", "interface": "deploy"}]' \
       cloudcomp100

Memory burn-in
^^^^^^^^^^^^^^

Available options (following the ``agent_burnin_`` + stress-ng stressor
(``vm``) + stress-ng option schema):

- ``agent_burnin_vm_timeout`` (default: 24 hours)
- ``agent_burnin_vm_vm-bytes`` (default: 98%)

to limit the overall runtime and to set the fraction of RAM to stress.

For instance, to limit the memory burn-in to 1 hour and the amount of RAM
to be used to 75%:

.. code-block:: console

   admin# openstack baremetal node set \
       --driver-info agent_burnin_vm_timeout=3600 cloudcomp100
   admin# openstack baremetal node set \
       --driver-info agent_burnin_vm_vm-bytes=75% cloudcomp100

Then launch the test:

.. code-block:: console

   admin# openstack baremetal node clean \
       --clean-steps '[{"step": "burnin_memory", "interface": "deploy"}]' \
       cloudcomp100

Disk burn-in
^^^^^^^^^^^^

Available options (following the ``agent_burnin_`` + fio stressor
(``fio_disk``) + fio option schema):

- ``agent_burnin_fio_disk_runtime`` (default: 0, meaning no time limit)
- ``agent_burnin_fio_disk_loops`` (default: 4)

to set the time limit and the number of iterations when going over the
disks.

For instance, to limit the number of loops to 2:

.. code-block:: console

   admin# openstack baremetal node set \
       --driver-info agent_burnin_fio_disk_loops=2 cloudcomp100

To launch a parallel SMART self-test on all devices after the disk burn-in
(which will fail the step if any of the tests fail), also set:

.. code-block:: console

   admin# openstack baremetal node set \
       --driver-info agent_burnin_fio_disk_smart_test=True cloudcomp100

Then launch the test:

.. code-block:: console

   admin# openstack baremetal node clean \
       --clean-steps '[{"step": "burnin_disk", "interface": "deploy"}]' \
       cloudcomp100

Network burn-in
^^^^^^^^^^^^^^^

The network test needs a pair of nodes, one acting as ``writer`` and the
other as ``reader``. The pairing can be defined either statically, i.e.
pairs are defined upfront, or dynamically via a distributed coordination
backend which orchestrates the pair matching. The static approach is more
predictable in terms of which nodes test each other; the dynamic approach
avoids nodes being blocked if one of the pair has problems, by simply
pairing all available nodes.

The |project_name| deployment does not run a coordination backend (such as
ZooKeeper), so network burn-in pairs are defined **statically**:

.. code-block:: console

   admin# openstack baremetal node set \
       --driver-info agent_burnin_fio_network_config='{"role": "writer", "partner": "cloudcomp101"}' \
       cloudcomp100
   admin# openstack baremetal node set \
       --driver-info agent_burnin_fio_network_config='{"role": "reader", "partner": "cloudcomp100"}' \
       cloudcomp101

The test also has a runtime option, which only needs to be set on the
writer:

.. code-block:: console

   admin# openstack baremetal node set \
       --driver-info agent_burnin_fio_network_runtime=600 cloudcomp100

The network burn-in is launched on **both** nodes of the pair:

.. code-block:: console

   admin# openstack baremetal node clean \
       --clean-steps '[{"step": "burnin_network", "interface": "deploy"}]' \
       cloudcomp100
   admin# openstack baremetal node clean \
       --clean-steps '[{"step": "burnin_network", "interface": "deploy"}]' \
       cloudcomp101

Both nodes wait for the other node to show up and block while waiting. If
the partner does not show up, the cleaning timeout will step in.

GPU burn-in
^^^^^^^^^^^

The GPU burn-in tests come in two parts:

- Check that the correct number of GPUs are visible to the operating
  system (only performed if ``agent_burnin_gpu_count`` is set to a value
  above 0)
- GPU burn-in test using gpu-burn

Available options (following the ``agent_burnin_`` + gpu stressor
(``gpu``) option schema):

- ``agent_burnin_gpu_install_dir`` (default: /opt/gpu-burn)
- ``agent_burnin_gpu_timeout`` (default: 24 hours)
- ``agent_burnin_gpu_memory`` (default: 95%)
- ``agent_burnin_gpu_count`` (default: 0, the GPU count check is disabled)

For instance, to limit the GPU burn-in to 10 minutes:

.. code-block:: console

   admin# openstack baremetal node set \
       --driver-info agent_burnin_gpu_timeout=600 cloudcomp100

Then launch the test:

.. code-block:: console

   admin# openstack baremetal node clean \
       --clean-steps '[{"step": "burnin_gpu", "interface": "deploy"}]' \
       cloudcomp100

.. note::

   The current |project_name| bare metal fleet does not include GPU nodes,
   so this test is not normally used. It is available for any future
   hardware where GPUs are passed through to the Ironic node.

Monitoring progress
-------------------

While the test runs, the node is in the ``cleaning`` (or ``clean wait``)
provision state. Progress can be watched with:

.. code-block:: console

   admin# watch -n 30 openstack baremetal node show cloudcomp100 \
       -f value -c provision_state -c last_clean_start

The test can be aborted at any time with:

.. code-block:: console

   admin# openstack baremetal node abort cloudcomp100

Multiple tests can be launched in a single call and are run in sequence:

.. code-block:: console

   admin# openstack baremetal node clean \
       --clean-steps '[{"step": "burnin_cpu", "interface": "deploy"}, \
                       {"step": "burnin_memory", "interface": "deploy"}]' \
       cloudcomp100

Adding ``--fast-track`` to the ``clean`` call keeps the node up between
consecutive clean calls, which shortens the overall time when several
burn-in steps are run one after another.

When the burn-in completes, the node returns to the state it was in before
the clean was launched (``manageable`` or ``available``) and is ready for
the next stage of the node life cycle (see :ref:`ironic-node-lifecycle`).

Collecting the results
----------------------

Most of the burn-in steps also report on the performance of the stressed
components, which is useful for verification or acceptance purposes. By
default, the output of the burn-in tools goes to the journal of the Ironic
Python Agent and is sent back to ironic-conductor as a log archive; on the
|project_name| deployment control plane logs are aggregated to OpenSearch
(see the Monitoring Services section of :doc:`access_to_services`).

To store the output of individual steps in files in the ramdisk instead
(from where they can be picked up by a logging pipeline), set one of the
``agent_burnin_cpu_outputfile``, ``agent_burnin_vm_outputfile``,
``agent_burnin_fio_disk_outputfile`` or ``agent_burnin_fio_network_outputfile``
parameters on the node:

.. code-block:: console

   admin# openstack baremetal node set \
       --driver-info agent_burnin_cpu_outputfile='/var/log/burnin.cpu' \
       cloudcomp100

.. note::

   Options set with ``openstack baremetal node set`` are not persisted in
   the |kayobe_config| repository and are lost if the node is re-provisioned
   through Kayobe (which regenerates ``driver-info`` from the inventory).
   For a standard acceptance test this is normally fine; if a particular
   configuration needs to be reapplied to many nodes, store the
   ``agent_burnin_*`` options in the node's host vars under
   ``etc/kayobe/inventory/host_vars/<node>/ironic`` in the |kayobe_config|
   repository instead.
