--------------

Brocade Openstack VDX Plugin SVI(L3) Networking
===============================================

This describes the setup of Openstack Plugins for Brocade VDX devices
for L3/SVI networking
https://github.com/openstack/networking-brocade/tree/master/networking\_brocade/vdx
|Setup of Openstack Plugin|

--------------

Fig 1. Setup of VDX Fabric with Compute Nodes

The figure(fig 1) shows a typical Physical deployment of Servers(Compute
Nodes) connected to VDX L2 Fabric.

-  eth1 on the controller Node is connected to VDX interface (e.g Te
   135/0/10)
-  eth1 on the compute Node is connected to VDX interface (e.g Te
   136/0/10)
-  NIC (eth1) on the servers (controller,compute ) are part of OVS
   bridge br1.

Note: To create bridge br1 on compute Nodes and add port eth1 to it.
sudo ovs-vsctl add-br br1 sudo ovs-vsctl add-port br1 eth1

There are two networks , GREEN(10.0.0.0/24) and RED(9.0.0.0/24). Virtual
Machines are created on both of these networks on each of the hosts. In
this setup, we would try to establish routing across the two networks
using Brocade L3/SVI Plugin.

Setup of Openstack Plugin
-------------------------

L3/SVI Networking can be setup on top of either L2(with AMPP Support) or
L2(without AMPP support). Please refer to the L2 Networking setup guides

Openstack Configurations (L3/SVI Setup)
---------------------------------------

Add the following line in '/etc/neutron/neutron.conf' to enable Brocade
SVI Plugin

::

    service_plugins = networking_brocade.vdx.non_ampp.ml2driver.l3_router_plugin.BrocadeSVIPlugin

Following **additional** configuration lines needs to be added to either
'/etc/neutron/plugins/ml2/ml2\_conf\_brocade.ini' or
'/etc/neutron/plugins/ml2/ml2\_conf.ini'. If added to
'/etc/neutron/plugins/ml2/ml2\_conf\_brocade.ini' then this file should
be given as config parameter during neutron-server startup.

::

    [svi]
    #List of rbridges on which 
    rbridge_ids=135,136
    is_vrf_required = True
    #enable L3 redundancy if needed by default redundancy is disabled
    redundancy=enabled
    vrrp_version = 2
    vrrp_group_id = 100
    vrrp_advertisement_interval = 5

Here [svi]

-  rbridge\_ids - list of rbridges on which Virtual Routing instances
   would be created.
-  redundancy - Is to be set to enabled if Virtual Routing instances
   have to be created on multiple rbridges.

Note : [ml2\_brocade] configuration lines needs to be added as per the
L2 Setup (AMPP or Non AMPP)

Openstack CLI Comands
---------------------

Create Networks
~~~~~~~~~~~~~~~

Create a GREEN Network (10.0.0.0/24) using neutron CLI's. Note down the
id of the network created which will be used during subsequent nova boot
commands.

::

    user@controller:~$ neutron net-create GREEN_NETWORK
    user@controller:~$ neutron subnet-create GREEN_NETWORK 10.0.0.0/24 --name GREEN_SUBNET --gateway=10.0.0.1
    user@controller:~$ neutron net-show GREEN_NETWORK
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | availability_zone_hints   |                                      |
    | availability_zones        | nova                                 |
    | created_at                | 2016-04-12T09:38:45                  |
    | description               |                                      |
    | id                        | d5c94db7-9040-481c-b33c-252618fb71f8 |
    | ipv4_address_scope        |                                      |
    | ipv6_address_scope        |                                      |
    | mtu                       | 1500                                 |
    | name                      | GREEN_NETWORK                        |
    | port_security_enabled     | True                                 |
    | provider:network_type     | vlan                                 |
    | provider:physical_network | physnet1                             |
    | provider:segmentation_id  | 12                                   |
    | router:external           | False                                |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   | 1217d77d-2638-4c5c-9777-f5cd4f4e5045 |
    | tags                      |                                      |
    | tenant_id                 | ed2196b380214e6ebcecc7d70e01eba4     |
    | updated_at                | 2016-04-12T09:38:45                  |
    +---------------------------+--------------------------------------+

Create a RED Network (9.0.0.0/24) using neutron CLI's. Note down the id
of the network created which will be used during subsequent nova boot
commands.

::

    user@controller:~$ neutron net-create RED_NETWORK
    user@controller:~$ neutron subnet-create RED_NETWORK 9.0.0.0/24 --name RED_SUBNET --gateway=9.0.0.1
    user@controller:~$ neutron net-show RED_NETWORK
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | availability_zone_hints   |                                      |
    | availability_zones        | nova                                 |
    | created_at                | 2016-04-12T09:39:53                  |
    | description               |                                      |
    | id                        | c994f6a1-5629-4617-b2af-64be34a744ec |
    | ipv4_address_scope        |                                      |
    | ipv6_address_scope        |                                      |
    | mtu                       | 1500                                 |
    | name                      | RED_NETWORK                          |
    | port_security_enabled     | True                                 |
    | provider:network_type     | vlan                                 |
    | provider:physical_network | physnet1                             |
    | provider:segmentation_id  | 33                                   |
    | router:external           | False                                |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   | 392fd70e-0c04-44db-8e4f-d5d6f4e1c09b |
    | tags                      |                                      |
    | tenant_id                 | ed2196b380214e6ebcecc7d70e01eba4     |
    | updated_at                | 2016-04-12T09:39:53                  |
    +---------------------------+--------------------------------------+

Check the availability Zones, We will launch one VM each on one of the
servers.

::

    user@controller:~$ nova availability-zone-list
    +-----------------------+----------------------------------------+
    | Name                  | Status                                 |
    +-----------------------+----------------------------------------+
    | internal              | available                              |
    | |- controller         |                                        |
    | | |- nova-conductor   | enabled :-) 2016-04-11T05:10:06.000000 |
    | | |- nova-scheduler   | enabled :-) 2016-04-11T05:10:07.000000 |
    | | |- nova-consoleauth | enabled :-) 2016-04-11T05:10:07.000000 |
    | nova                  | available                              |
    | |- compute            |                                        |
    | | |- nova-compute     | enabled :-) 2016-04-11T05:10:10.000000 |
    | |- controller         |                                        |
    | | |- nova-compute     | enabled :-) 2016-04-11T05:10:05.000000 |
    +-----------------------+----------------------------------------+

Launching Virtual Machines
~~~~~~~~~~~~~~~~~~~~~~~~~~

Boot VM1 on Server by the name "controller"

::

    user@controller:~$nova boot --nic net-id=$(neutron net-list | awk '/GREEN_NETWORK/ {print $2}') 
     --image cirros-0.3.4-x86_64-uec --flavor m1.tiny --availability-zone nova:controller VM1

Boot VM2 on Server by the name "compute"

::

    user@controller:~$nova boot --nic net-id=$(neutron net-list | awk '/GREEN_NETWORK/ {print $2}')
     --image cirros-0.3.4-x86_64-uec --flavor m1.tiny --availability-zone nova:compute VM2

Boot VM3 on Server by the name "controller"

::

    user@controller:~$nova boot --nic net-id=$(neutron net-list | awk '/RED_NETWORK/ {print $2}') 
     --image cirros-0.3.4-x86_64-uec --flavor m1.tiny --availability-zone nova:controller VM3

Boot VM4 on Server by the name "compute"

::

    user@controller:~$nova boot --nic net-id=$(neutron net-list | awk '/RED_NETWORK/ {print $2}') 
     --image cirros-0.3.4-x86_64-uec --flavor m1.tiny --availability-zone nova:compute VM4

Create a Router
~~~~~~~~~~~~~~~

Create a Router instance having both the networks (GREEN\_NETWORK and
RED\_NETWORK)

::

    neutron router-create demo-router
    neutron router-interface-add demo-router GREEN_SUBNET
    neutron router-interface-add demo-router RED_SUBNET

VDX
~~~

Following Routing instances would have created on VDX

::

    sw0# show running-config rbridge-id 135 vrf
    rbridge-id 135
     vrf openstack-vrf-b076cce6-299d-4499
      rd 0766:0766
      address-family ipv4 unicast
      !
     !
     vrf test
      rd 10:10
     !
    !

    sw0# show running-config rbridge-id 135 interface ve
    rbridge-id 135
     interface Ve 12
      vrf forwarding openstack-vrf-b076cce6-299d-4499
      ip proxy-arp
      ip address 10.0.0.5/24
      vrrp-group 100 version 2
       virtual-ip 10.0.0.1
       advertisement-interval 5
       enable
       preempt-mode
       priority 1
      !
      no shutdown
     !
     interface Ve 33
      vrf forwarding openstack-vrf-b076cce6-299d-4499
      ip proxy-arp
      ip address 9.0.0.6/24
      vrrp-group 100 version 2
       virtual-ip 9.0.0.1
       advertisement-interval 5
       enable
       preempt-mode
       priority 1
      !
      no shutdown
     !
    !

Ping between Virtual Machines across Networks
---------------------------------------------

We should now be able to ping between Virtual Machines across Networks
(GREEN\_Network and RED\_NETWORK)

.. |Setup of Openstack Plugin| image:: https://2.bp.blogspot.com/-tw3rvPCXtqE/Vv4Da2mvleI/AAAAAAAADiI/9GJGVCirmUkFsVhWGNtA15zEf-9xt4n6A/s400/L2+Fabric+Image.png

