Deploying Manila in the Overcloud
=================================

This guide assumes that your undercloud is already installed and ready to
deploy an overcloud with Manila enabled.

Deploying the Overcloud with the Internal Ceph Backend
------------------------------------------------------
Ceph deployed by TripleO can be used as a Manila share backend. Make sure that
Ceph, Ceph MDS and Manila Ceph environment files are included when deploying the
Overcloud::

    openstack overcloud deploy --templates \
    -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-mds.yaml \
    -e /usr/share/openstack-tripleo-heat-templates/environments/manila-cephfsnative-config.yaml

.. note::
   These and any other environment files or options passed to the overcloud
   deploy command, are referenced below as the "full environment". We assumed
   the ``--plan`` flag is not what we want to use for this example.

Network Isolation
~~~~~~~~~~~~~~~~~
When mounting a ceph share from a user instance, the user instance needs access
to the Ceph public network. When mounting a ceph share from a user instance,
the user instance needs access to the Ceph public network, which in TripleO
maps to the Overcloud storage network.  In an Overcloud which uses isolated
networks the tenant network and storage network are isolated from one another
so user instances cannot reach the Ceph public network unless the cloud
administrator creates a provider network in neutron that maps to the storage
network and exposes access to it.

Before deploying Overcloud make sure that there is a bridge for storage network
interface. If single NIC with VLANs network configuration is used (as in
``/usr/share/openstack-tripleo-heat-templates/network/config/single-nic-vlans/``)
then by default ``br-ex`` bridge is used for storage network and no additional
customization is required for Overcloud deployment. If a dedicated interface is
used for storage network (as in
``/usr/share/openstack-tripleo-heat-templates/network/config/multiple-nics/``)
then update storage interface for each node type (controller, compute, ceph) to
use bridge. The following interface definition::

    - type: interface
        name: nic2
        use_dhcp: false
        addresses:
        - ip_netmask:
            get_param: StorageIpSubnet

should be replaced with::

    - type: ovs_bridge
      name: br-storage
      use_dhcp: false
      addresses:
      - ip_netmask:
          get_param: StorageIpSubnet
      members:
      - type: interface
        name: nic2
        use_dhcp: false
        primary: true

And pass following parameters when deploying Overcloud to allow Neutron to map
provider networks to the storage bridge::

      parameter_defaults:
          NeutronBridgeMappings: datacentre:br-ex,storage:br-storage
          NeutronFlatNetworks: datacentre,storage

If the storage network uses VLAN, include storage network in
``NeutronNetworkVLANRanges`` parameter. For example::

    NeutronNetworkVLANRanges: 'datacentre:100:1000,storage:30:30'

.. warning::
    If network isolation is used, make sure that storage provider network
    subnet doesn't overlap with IP allocation pool used for Overcloud storage
    nodes (controlled by ``StorageAllocationPools`` heat parameter).
    ``StorageAllocationPools`` is by default set to
    ``[{'start': '172.16.1.4', 'end': '172.16.1.250'}]``. It may be necessary
    to shrink this pool, for example::

        StorageAllocationPools: [{'start': '172.16.1.4', 'end': '172.16.1.99'}]

When Overcloud is deployed, create a provider network which can be used to
access storage network.

* If single NIC with VLANs is used, then the provider network is mapped
  to the default datacentre network::

      neutron net-create storage --shared --provider:physical_network \
        datacentre --provider:network_type vlan --provider:segmentation_id 30

      neutron subnet-create --name storage-subnet \
        --allocation-pool start=172.16.1.100,end=172.16.1.120 \
        --enable-dhcp storage 172.16.1.0/24

* If a custom bridge was used for storage network interface (``br-storage`` in
  the example above) then provider network is mapped to the network specified
  by ``NeutronBridgeMappings`` parameter (``storage`` network in the example
  above)::

      neutron net-create storage --shared --provider:physical_network storage \
        --provider:network_type flat

      neutron subnet-create --name storage-subnet \
        --allocation-pool start=172.16.1.200,end=172.16.1.220 --enable-dhcp \
        storage 172.16.1.0/24 --no-gateway

.. note::
    Allocation pool should not overlap with storage network
    pool used for storage nodes (``StorageAllocationPools`` parameter).
    You may also need to shrink storage nodes pool size to reserve more IPs
    for tenants using the provider network.

.. note::

    Make sure that subnet CIDR matches storage network CIDR (``StorageNetCidr``
    parameter)and
    segmentation_id matches VLAN ID for the storage network traffic
    (``StorageNetworkVlanID`` parameter).

Then Ceph shares can be accessed from a user instance by adding the provider
network to the instance.

.. note::

    Cloud-init by default configures only first network interface to use DHCP
    which means that user instances will not have network interface for storage
    network autoconfigured. You can configure it manually or use
    `dhcp-all-interfaces <https://docs.openstack.org/diskimage-builder/elements/dhcp-all-interfaces/README.html>`_.


Deploying Manila in the overcloud with CephFS through NFS and a composable network
----------------------------------------------------------------------------------

The CephFS through NFS back end is composed of Ceph metadata servers (MDS),
NFS Ganesha (the NFS gateway), and the Ceph cluster service components.
The manila CephFS NFS driver uses NFS-Ganesha gateway to provide NFSv4 protocol
access to CephFS shares.
The Ceph MDS service maps the directories and file names of the file system
to objects that are stored in RADOS clusters.
The NFS-Ganesha service runs on the Controller nodes with the Ceph services.


CephFS with NFS-Ganesha deployment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

CephFS through NFS deployments use an extra isolated network, StorageNFS.
This network is deployed so users can mount shares over NFS on that network
without accessing the Storage or Storage Management networks which are
reserved for infrastructure traffic.

The ControllerStorageNFS custom role configures the isolated StorageNFS network.
This role is similar to the default `Controller.yaml` role file with the addition
of the StorageNFS network and the CephNfs service, indicated by the `OS::TripleO::Services:CephNfs`
service.


#. To create the StorageNFSController role, used later in the process by the
   overcloud deploy command, run::

    openstack overcloud roles generate --roles-path /usr/share/openstack-tripleo-heat-templates/roles \
      -o /home/stack/roles_data.yaml ControllerStorageNfs Compute CephStorage

#. Run the overcloud deploy command including the new generated `roles_data.yaml`
   and the `network_data_ganesha.yaml` file that will trigger the generation of
   this new network. The final overcloud command must look like the following::

     openstack overcloud deploy \
       --templates /usr/share/openstack-tripleo-heat-templates  \
       -n /usr/share/openstack-tripleo-heat-templates/network_data_ganesha.yaml \
       -r /home/stack/roles_data.yaml \
       -e /home/stack/containers-default-parameters.yaml   \
       -e /usr/share/openstack-tripleo-heat-templates/environments/network-isolation.yaml   \
       -e /home/stack/network-environment.yaml  \
       -e/usr/share/openstack-tripleo-heat-templates/environments/cephadm/cephadm.yaml  \
       -e /usr/share/openstack-tripleo-heat-templates/environments/cephadm/ceph-mds.yaml  \
       -e /usr/share/openstack-tripleo-heat-templates/environments/manila-cephfsganesha-config.yaml


.. note::

  The network_data_ganesha.yaml file contains an additional section that defines
  the isolated StorageNFS network. Although the default settings work for most
  installations, you must edit the YAML file to add your network settings,
  including the VLAN ID, subnet, and other settings::

    name: StorageNFS
    enabled: true
    vip: true
    name_lower: storage_nfs
    vlan: 70
    ip_subnet: '172.16.4.0/24'
    allocation_pools: [{'start': '172.16.4.4', 'end': '172.16.4.149'}]
    ipv6_subnet: 'fd00:fd00:fd00:7000::/64'
    ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:7000::10', 'end': 'fd00:fd00:fd00:7000:ffff:ffff:ffff:fffe'}]


Configure the StorageNFS network
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

After the overcloud deployment is over, create a corresponding `StorageNFSSubnet` on
the neutron-shared provider network.
The subnet is the same as the storage_nfs network definition in the `network_data_ganesha.yml`
and ensure that the allocation range for the StorageNFS subnet and the corresponding
undercloud subnet do not overlap.

.. note::

  No gateway is required because the StorageNFS subnet is dedicated to serving NFS shares

In order to create the storage_nfs subnet, run::

  openstack subnet create --allocation-pool start=172.16.4.150,end=172.16.4.250 \
    --dhcp --network StorageNFS --subnet-range 172.16.4.0/24 \
    --gateway none StorageNFSSubnet

#. Replace the `start=172.16.4.150,end=172.16.4.250` IP values with the IP
   values for your network.
#. Replace the `172.16.4.0/24` subnet range with the subnet range for your
   network.


Deploying the Overcloud with an External Backend
------------------------------------------------
.. note::

    The :doc:`../deployment/template_deploy` doc has a more detailed explanation of the
    following steps.

#. Copy the Manila driver-specific configuration file to your home directory:

   - Dell-EMC Isilon driver::

       sudo cp /usr/share/openstack-tripleo-heat-templates/environments/manila-isilon-config.yaml ~

   - Dell-EMC Unity driver::

       sudo cp /usr/share/openstack-tripleo-heat-templates/environments/manila-unity-config.yaml ~

   - Dell-EMC Vmax driver::

       sudo cp /usr/share/openstack-tripleo-heat-templates/environments/manila-vmax-config.yaml ~

   - Dell-EMC VNX driver::

       sudo cp /usr/share/openstack-tripleo-heat-templates/environments/manila-vnx-config.yaml ~

   - NetApp driver::

       sudo cp /usr/share/openstack-tripleo-heat-templates/environments/manila-netapp-config.yaml ~

#. Edit the permissions (user is typically ``stack``)::

    sudo chown $USER ~/manila-*-config.yaml
    sudo chmod 755 ~/manila-*-config.yaml

#. Edit the parameters in this file to fit your requirements.

   - Fill in or override the values of parameters for your back end.

   - Since you have copied the file out of its original location,
     replace relative paths in the resource_registry with absolute paths
     based on ``/usr/share/openstack-tripleo-heat-templates``.

#. Continue following the TripleO instructions for deploying an overcloud.
   Before entering the command to deploy the overcloud, add the environment
   file that you just configured as an argument. For example::

    openstack overcloud deploy --templates \
      -e <full environment> -e ~/manila-[isilon or unity or vmax or vnx or netapp]-config.yaml

#. Wait for the completion of the overcloud deployment process.


Creating the Share
------------------

.. note::

    The following steps will refer to running commands as an admin user or a
    tenant user. Sourcing the ``overcloudrc`` file will authenticate you as
    the admin user. You can then create a tenant user and use environment
    files to switch between them.

#. Create a share network to host the shares:

   - Create the overcloud networks. The :doc:`../deployment/install_overcloud`
     doc has a more detailed explanation about creating the network
     and subnet. Note that you may also need to perform the following
     steps to get Manila working::

       neutron router-create router1
       neutron router-interface-add router1 [subnet id]

   - List the networks and subnets [tenant]::

       neutron net-list && neutron subnet-list

   - Create a share network (typically using the private default-net net/subnet)
     [tenant]::

       manila share-network-create --neutron-net-id [net] --neutron-subnet-id [subnet]

#. Create a new share type (yes/no is for specifying if the driver handles
   share servers) [admin]::

    manila type-create [name] [yes/no]

#. Create the share [tenant]::

    manila create --share-network [share net ID] --share-type [type name] [nfs/cifs] [size of share]


Accessing the Share
-------------------

#. To access the share, create a new VM on the same Neutron network that was
   used to create the share network::

    nova boot --image [image ID] --flavor [flavor ID] --nic net-id=[network ID] [name]

#. Allow access to the VM you just created::

    manila access-allow [share ID] ip [IP address of VM]

#. Run ``manila list`` and ensure that the share is available.

#. Log into the VM::

    ssh [user]@[IP]

.. note::

    You may need to configure Neutron security rules to access the
    VM. That is not in the scope of this document, so it will not be covered
    here.

5. In the VM, execute::

    sudo mount [export location] [folder to mount to]

6. Ensure the share is mounted by looking at the bottom of the output of the
   ``mount`` command.

7. That's it - you're ready to start using Manila!
