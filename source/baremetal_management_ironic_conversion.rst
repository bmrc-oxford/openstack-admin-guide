.. include:: vars.rst

=============================
Dynamic Hypervisor Conversion
=============================

This section describes procedures for changing the role of nodes in the
cluster. This includes converting a bare metal node into a hypervisor and
vice versa.

A hypervisor created by these procedures is a virtual machine launched on a
bare metal node that is managed by Ironic. The virtual machine runs a full
KVM hypervisor (nova compute), and tenant instances are scheduled onto it in
the same way as onto any other hypervisor. Because the underlying bare metal
node remains Ironic-managed, it can later be converted back to a general
purpose bare metal node when hypervisor capacity is no longer needed.

.. note::
    It is recommended that this process is carried out inside of ``tmux`` as
    one window can be used for the OpenStack client and a second window for
    Kayobe commands.
    The OpenStack client must be set up, making sure to install the ironic
    client and acquire admin credentials.
    To set up a Kayobe environment please review :doc:`working_with_kayobe`.

Hypervisor conversion
=====================

This procedure will enable the conversion of bare metal nodes into
hypervisors. This would be required if the current number of hypervisors is
not meeting demand. The steps outlined below cover all required changes and
commands within OpenStack and the Kayobe config.

OpenStack commands
------------------

#. Source the OpenStack admin credentials::

    source admin-openrc.sh

#. Choose a node to convert. Make note of the desired node's resource
   class::

    openstack baremetal node show <node> -f value -c resource_class | awk '{print tolower($0)}'

#. Check if there is a Nova instance currently running on the node you wish
   to convert::

    openstack baremetal node show <node> -f value -c instance_uuid

#. If an instance is present, check if it can be safely deleted, and delete
   it if so. If not, pick another node::

    openstack server delete <instance_uuid>

#. Wait for the node to finish deprovisioning and cleaning. It should end in
   the ``available`` provision state. You can check the state using
   ``baremetal node show``::

    openstack baremetal node show <node> -f value -c provision_state

#. Set the active project to the hypervisors project::

    export OS_PROJECT_NAME=|hypervisor_project_name|
    unset OS_PROJECT_ID

#. Launch a server onto the bare metal with the ``overcloud image``, on
   |workload_provision_network_name|, accessible with the hypervisor SSH
   key; the flavor is taken from step 2::

    openstack server create --flavor <node_resource_class> --image <overcloud_image> --network |workload_provision_network_name| --key-name <key_name> --use-config-drive <node>-hypervisor

#. Acquire the IP address that Neutron has assigned the hypervisor
   instance::

    openstack server show <node>-hypervisor -f value -c addresses

Kayobe commands
---------------

#. Add the hypervisor to the appropriate group within the inventory hosts
   file. In the |project_name| configuration hosts are listed in the
   base-layer file ``etc/kayobe/inventory/hosts``::

    [compute_ironic_dell_hypervisor]
    <node>-hypervisor

.. note::
    You may need to create the group that matches the resource class if one
    is not already present. The new group must be added to the
    ``[compute:children]`` entry in ``etc/kayobe/inventory/groups`` so that
    the host is picked up as a compute host for the overcloud deployment.

#. Apply the static physical network configuration of the switches in the
   EVPN fabric. The |project_name| deployment manages its Cumulus switches
   with ``kayobe physical network configure`` (see
   :ref:`static-switch-config`), so no vendor-specific workaround playbook
   is required. Preview the changes first::

    kayobe physical network configure --group switches --display

   If the output looks correct, apply it::

    kayobe physical network configure --group switches

.. note::
    NGS continues to manage the dynamic per-port VLAN membership of the bare
    metal node (see :ref:`dynamic-switch-configuration`). This step only
    applies the static base configuration (trunks, MTU and EVPN settings) to
    the switches.

#. Add the IP address that Neutron assigned the hypervisor to
   ``${KAYOBE_CONFIG_PATH}/environments/${KAYOBE_ENVIRONMENT}/network-allocation.yml``::

    provision_wl_net_ips:
        <node>-hypervisor: <hypervisor_ip>

#. At this point the hypervisor may be reachable over SSH via the
   provision_wl IP::

    ssh cloud-user@<hypervisor_ip>

.. note::
    If the hypervisor is not reachable check the build status of the instance
    with ``openstack baremetal node show <node> -f value -c provision_state``.

#. Perform host configuration of the hypervisor::

    kayobe overcloud host configure --limit <node>-hypervisor

#. Perform service deploy of the hypervisor::

    kayobe overcloud service deploy -kl <node>-hypervisor
    kayobe overcloud service deploy -kl monitoring -kt prometheus

.. note::
    After the service deploy has finished the hypervisor should be ready for
    use. You can attempt to launch instances on the hypervisor using
    ``openstack server create --image <image_name> --flavor <flavor_name> --network <network_name> --key-name <key_name> --availability-zone nova::<node>-hypervisor <node>-hypervisor-test-vm``.

#. Commit the changes to the |kayobe_config| repository and open a pull
   request::

    git checkout -b add-<node>-hypervisor
    git add -u
    git commit -m "feat: convert baremetal to hypervisor" -m "converted: <node> to Ironic controlled hypervisor"
    git push -u origin add-<node>-hypervisor

Baremetal conversion
====================

This procedure will convert hypervisors back into bare metal nodes,
effectively reverting the procedure outlined above. This would be required
if the demand for hypervisors has dropped, allowing for fewer hypervisors to
be needed. The steps outlined below cover all required changes and commands
within the Kayobe config and OpenStack.

Kayobe commands
---------------

#. Source the OpenStack admin credentials::

    source admin-openrc.sh

#. Disable the nova compute services on the hypervisor you are converting
   into a bare metal node::

    kayobe playbook run ${KAYOBE_CONFIG_PATH}/ansible/maintenance/nova-compute-disable.yml --limit <node>-hypervisor

#. Drain the hypervisor of all virtual machines::

    kayobe playbook run ${KAYOBE_CONFIG_PATH}/ansible/maintenance/nova-compute-drain.yml --limit <node>-hypervisor

#. Delete the hypervisor from the inventory hosts file::

    sed -e '/<node>-hypervisor/d' -i ${KAYOBE_CONFIG_PATH}/inventory/hosts

#. Remove all IP allocations for the hypervisor from ``network-allocation.yml``::

    sed -e '/<node>-hypervisor/d' -i ${KAYOBE_CONFIG_PATH}/environments/${KAYOBE_ENVIRONMENT}/network-allocation.yml

#. Perform a limited service deploy focusing on ``prometheus`` within the
   controllers::

    kayobe overcloud service deploy -kl monitoring -kt prometheus

#. Commit the changes to the |kayobe_config| repository and open a pull
   request::

    git checkout -b remove-<node>-hypervisor
    git add -u
    git commit -m "feat: convert hypervisor to baremetal" -m "converted: <node> back to generalist baremetal"
    git push -u origin remove-<node>-hypervisor

OpenStack commands
------------------

#. Source the OpenStack admin credentials::

    source admin-openrc.sh

#. Delete the hypervisor instance running on the bare metal::

    openstack server delete <node>-hypervisor

#. Wait for the node to finish deprovisioning and to move through the
   process to an ``available`` provision state. You can check the state
   using ``baremetal node show``::

    watch -n 10 -d=permanent openstack baremetal node show <node> \
        -f yaml -c provision_state -c provision_updated_at

#. Remove the OpenStack compute service from the disabled and drained
   hypervisor::

    openstack compute service delete <node>-hypervisor

.. note::
    The |project_name| deployment uses OVS, so there are no OVN controller
    or metadata agents to delete manually, as would be required on an OVN
    deployment. The agent registrations of the retired hypervisor are
    cleaned up by Neutron once they time out.
