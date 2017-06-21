# OpenStack Ocata Installation Manual
This manual explains how to install OpenStack Ocata on a cluster. It leverages the [__official installation guide of OpenStack Ocata__](https://docs.openstack.org/ocata/install-guide-ubuntu/) with customized networking services and some corrections. 

#### Table of Content

* [Deployment Scenario](#scenario)
* [Environment](#envi)
    * [Configure Dell N2024 Switch](#switch)
    * [Host Networking](#host-net)
    * [Other Configurations](#other-config)
* [Identity Service (Keystone)](#keystone)
* [Image Service (Glance)](#glance)
* [Compute Service (Nova)](#nova)
    * [Controller Node](#nova-ctl)
	* [Compute Node](#nova-cn)
* [Networking Service (Neutron)](#neutron)
	* [Controller Node](#neutron-ctl)
	* [Network Node](#neutron-nn)
	* [Compute Node](#neutron-cn)
* [Dashboard (Horizon)](#horizon)
* [Create Experiment Setup](#exp)
* [Reference](#ref)

## <a name="scenario"></a>Deployment Scenario
There are different ways to deploy OpenStack. This manual installs OpenStack modules on three types of nodes (_e.g._, physical servers.): controller node (one), network node (one), and compute node (multiple). The figure 1 shows OpenStack module installed on different types of nodes.

This deployment requires three logic networks: management network (connects controller node, network node and compute nodes), instance network (connects network node and compute nodes), external network (for external access of each node). In this deployment, we use external network as management network which is not recommended.

Number of NICs on different node:
* Controller node: 1 NIC
* Network node: 3 NICs
* Compute nodes: 2 NICs

![](https://github.com/lianjiecao/cap-cluster/blob/master/openstack-cap.png?raw=true)
Figure 1. OpenStack deployment illustration


## <a name="envi"></a>Environment

### <a name="switch"></a>Configure Dell N2024 switch
1. Connect the switch console port to a server (cap01) using RJ45-RS232 cable.
2. Log into the switch using ```screen```.

    ```# screen /dev/ttyS0 9600 ```
3. Configure VLAN.
    * Enter priviledge mode:
        ```
        sw-lwsnb132a> enable
        sw-lwsnb132a# config
        ```
    * Create VLAN with name ```openstack```:
        ```
        sw-lwsnb132a(config)# vlan 100
        sw-lwsnb132a(config-vlan100)# name openstack
        sw-lwsnb132a(config-vlan100)# exit
        sw-lwsnb132a(config)# vlan 100
        sw-lwsnb132a(config-vlan100)# name openstack
        sw-lwsnb132a(config-vlan100)# exit
        ```
        
    * Add port to VLAN (repeat to add all ports):
        ```
        sw-lwsnb132a(config)#vlan 100
        sw-lwsnb132a(config-vlan100)#name openstack
        sw-lwsnb132a(config-vlan100)#exit
        ```

### <a name="host-net"></a>Host Networking

* Edit ```/etc/network/interfaces``` to configure NICs. Note, the second NIC is not necessary for controller in our setup.

**Controller node and compute nodes**
```
# External NIC for access and management
auto eth0
iface eth0 inet static
    address 128.10.130.121
    netmask 255.255.255.0
    network 128.10.130.0
    broadcast 128.10.130.255
    gateway 128.10.130.250
    dns-nameservers 128.10.2.5 128.210.11.5
    dns-search cs.purdue.edu

# Overlay NIC for instance overlay network
auto eth1
iface eth1 inet static
    address 10.0.0.11
    netmask 255.255.255.0
    network 10.0.0.0
    broadcast 10.0.0.255
```

**Network node**

Note, the third NIC will be added to external OVS bridge to provide external connectivity for instances in Neutron configuration.
```
# External NIC for access and management
auto enp32s0
iface enp32s0 inet static
    address 128.10.130.122
    netmask 255.255.255.0
    network 128.10.130.0
    broadcast 128.10.130.255
    gateway 128.10.130.250
    # dns-* options are implemented by the resolvconf package, if installed
    dns-nameservers 128.10.2.5 128.210.11.5
    dns-search cs.purdue.edu

# Overlay NIC for instance overlay network
auto enp34s0
iface enp34s0 inet static
    address 10.0.0.21
    netmask 255.255.255.0
    network 10.0.0.0
    broadcast 10.0.0.255

# External NIC for external access of instances
auto enp16s0
iface enp16s0 inet manual
    up ip link set dev $IFACE up
    down ip link set dev $IFACE down
```

* Edit ```/etc/hosts``` to provide hostname resolution in management network on each node. In our deployment, there is a DNS server to resolve hostnames. So this step is not necessary.
    ```
    127.0.0.1        localhost
    128.10.130.121   cap01    # controller node
    128.10.130.122   cap02    # network node
    128.10.130.67    wabash   # compute node
    ```

### <a name="other-config"></a>Other Configurations
* Follow [<span style="text-decoration:underline">these steps</span>](https://docs.openstack.org/ocata/install-guide-ubuntu/environment.html) to finish NTP, OpenStack packages, SQL databse, message queue, and memcached configurations.


## <a name="keystone"></a>Identity Service (Keystone)
* Follow [<span style="text-decoration:underline">this</span>](https://docs.openstack.org/ocata/install-guide-ubuntu/keystone.html) to install Keystone on controller node.

## <a name="glance"></a>Image Service (Glance)
* Follow [<span style="text-decoration:underline">this</span>](https://docs.openstack.org/ocata/install-guide-ubuntu/glance.html) to install Glance on controller node.

## <a name="nova"></a>Compute Service (Nova)
* Follow [<span style="text-decoration:underline">this</span>](https://docs.openstack.org/ocata/install-guide-ubuntu/nova.html) to install Nova on controller node.

* Extra configurations:
    1. Enable ```--availability-zone``` when launching instances. This option helps users to create instance on a certain host/compute node. On controller node, open/create ```/etc/nova/policy.json``` and add the following line. This allows all users to use this option.
    
        ```
        {
            "os_compute_api:servers:create:forced_host":""
        }
        ```
    **Note, previous solutions like [<span style="text-decoration:underline">this</span>](http://www.googlinux.com/openstack-launching-an-instance-on-a-specific-compute-host/) is no longer working.**




## <a name="neutron"></a>Networking Service (Neutron)
Different from the official installation guide, this manual chooses Open vSwitch as the machinsm driver for Neutron. The table below shows Neutron modules to be installed on different types of nodes.
|   Controller node   |        Network node        |       Compute node       |
|:-------------------:|:--------------------------:|:------------------------:|
| neutron-server      | neutron-openvswitch-agent | neutron-openvswitch-agent |
| neutron-plugin-ml2  | neutron-l3-agent          |                           |
|                     | neutron-dhcp-agent        |                           |
|                     | neutron-metadata-agent    |                           |

**Note**, ```neutron-dhcp-agent``` and ```neutron-metadata-agent``` can be installed on compute nodes to improve HA as well.

### <a name="neutron-ctl"></a>Controller Node:

* Create Neutron user in database:
    ```bash
    sudo su
    mysql
    MariaDB [(none)]> CREATE DATABASE neutron;
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
    MariaDB [(none)]> exit
    ```

* Create Neutron roles and users in Keystone:
    ```bash
    . admin-openrc
    openstack user create --domain default --password-prompt neutron # NEURON_PASS
    openstack role add --project service --user neutron admin
    openstack service create --name neutron --description "OpenStack Networking" network
    openstack endpoint create --region RegionOne network public http://cap01:9696
    openstack endpoint create --region RegionOne network internal http://cap01:9696
    openstack endpoint create --region RegionOne network admin http://cap01:9696
    ```

* Install and configure Neutron components:
    ```bash
    sudo su
    apt install neutron-server neutron-plugin-ml2
    ```

1. Edit ```/etc/neutron/neutron.conf``` to enable database connection, ```RabbitMQ``` access, ```Keystone``` authentication, ```nova``` interaction, and ```ml2``` plugin.

    *Click [<span style="text-decoration:underline">here</span>](https://raw.githubusercontent.com/lianjiecao/cap-cluster/master/ctl-neutron.conf) for a example configuration file.
    
2. Edit ```/etc/neutron/plugins/ml2/ml2_conf.ini``` to configure type drivers (_e.g._, flat, vlan, vxlan and gre) and machanism drivers (_e.g._, linuxbridge, openvswitch and l2population). In this manual, openvswitch+vxlan is chosen for Neutron.

    * Configure type drivers and machanism drivers in ```[ml2]``` section.
        ```
        [ml2]
        type_drivers = flat,vlan,vxlan,gre
        tenant_network_types = vxlan    
        mechanism_drivers = openvswitch,l2population
        extension_drivers = port_security
        ```
    * Configure ```[ml2_type_flat]``` with the name of flat network to be created next.
        ```
        [ml2_type_flat]
        flat_networks = provider
        ```
    * Configure ```vni_ranges``` with values for VXLAN IDs. Here uses large number to better differentiate virtual VXLANs for OpenStack.
        ```
        [ml2_type_vxlan]
        vni_ranges = 65537:69999
        ```
    * Configure firewall and security group options in ```[securitygroup]```.
        ```
        [securitygroup]
        firewall_driver = iptables_hybrid
        enable_security_group = true
        enable_ipset = true
        ```
    Click [<span style="text-decoration:underline">here</span>]() for a complete configuration file.
    
3. Edit ```[neutron]``` section in ```/etc/nova/nova.conf``` configure Nova module to use Neutron networking service.
    ```
    [neutron]
    url = http://cap01:9696
    auth_url = http://cap01:35357
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    region_name = RegionOne
    project_name = service
    username = neutron
    password = NEURON_PASS
    service_metadata_proxy = true
    metadata_proxy_shared_secret = METADATA_SECRET
    ```

4. Restart ```neutron-server``` service.
    ```
    # service neutron-server restart
    ```

### <a name="neutron-nn"></a>Network Node:
1. Install neutron models.
    ```
    # apt install neutron-openvswitch-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
    ```

2. Edit ```/etc/neutron/neutron.conf``` to enable ```Keystone``` and ```RabbitMQ``` access and disable database connection.
    * Click [<span style="text-decoration:underline">here</span>](https://raw.githubusercontent.com/lianjiecao/cap-cluster/master/nn-neutron.conf) for a complete configuration file.

3. Edit ```/etc/neutron/plugins/ml2/openvswitch_agent.ini``` to configure Open vSwitch.
    * Configure ```[agent]``` section to use vxlan and l2_population.
        ```
        [agent]
        tunnel_types = vxlan
        l2_population = True
        arp_responder = True
        ```
    * Configure ```[ovs]``` with the IP address of the NIC for instance overlay network and OVS bridge name for provider/external network.
        ```
        [ovs]
        local_ip = 10.0.0.21
        bridge_mappings = provider:br-provider ### network_name:bridge_name
        ```
    * Configure firewall and security group the same way as ```/etc/neutron/plugins/ml2/ml2_conf.ini``` on controller node.
        ```
        [securitygroup]
        firewall_driver = iptables_hybrid
        enable_security_group = true
        enable_ipset = true
        ```
	* Create OVS bridge with the same name in ```bridge_mappings = provider:br-provider``` and add a external NIC to the bridge.
    	```
        # ovs-vsctl add-br br-provider
        # ovs-vsctl add-port br-provider enp16s0
        ```

4. Edit ```/etc/neutron/l3_agent.ini``` to configure Neutron L3 agent. ```external_network_bridge``` is left blank on purpose.
    ```
    [DEFAULT]
    interface_driver = openvswitch
    external_network_bridge =
    ```

5. Edit ```/etc/neutron/dhcp_agent.ini``` to configure Neutron DHCP agent. DHCP agent is implemented based on dnsmasq.
    ```
    [DEFAULT]
    interface_driver = openvswitch
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    enable_isolated_metadata = true
    ```

6. Edit ```/etc/neutron/metadata_agent.ini``` to configure Neutron Metadate agent.
    ```
    [DEFAULT]
    nova_metadata_ip = cap01
    metadata_proxy_shared_secret = METADATA_SECRET
    ```

7. Restart Neutron services.
    ```
    # service openvswitch-switch restart
    # service neutron-openvswitch-agent restart
    # service neutron-dhcp-agent restart
    # service neutron-metadata-agent restart
    # service neutron-l3-agent restart
    ```

### <a name="neutron-cn"></a>Compute Node:
1. Install neutron models.
    ```
    # apt install neutron-openvswitch-agent
    ```

2. Edit ```/etc/neutron/neutron.conf``` to enable ```Keystone``` and ```RabbitMQ``` access and disable database connection.
    * Click [<span style="text-decoration:underline">here</span>](https://raw.githubusercontent.com/lianjiecao/cap-cluster/master/cn-neutron.conf) for a complete configuration file.

3. Edit ```/etc/neutron/plugins/ml2/openvswitch_agent.ini``` to configure Open vSwitch. Very similar to the configuration on network node except ```bridge_mappings```.
    ```
    [agent]
    tunnel_types = vxlan
    l2_population = True
    arp_responder = True
    [ovs]
    local_ip = 10.0.0.31
    [securitygroup]
    firewall_driver = iptables_hybrid
    enable_security_group = true
    enable_ipset = true
    ```

4. Edit ```[neutron]``` section in ```/etc/nova/nova.conf``` configure Nova module to use Neutron networking service.
    ```
    [neutron]
    url = http://cap01:9696
    auth_url = http://cap01:35357
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    region_name = RegionOne
    project_name = service
    username = neutron
    password = NEURON_PASS
    ```

5. Restart Nova and Neutron service.
    ```
    # service nova-compute restart
    # service neutron-openvswitch-agent restart
    ```

## <a name="horizon"></a>Dashboard (Horizon)
**On controller node:**

1. Install Horizon module.

    ```# apt install openstack-dashboard```

2. Edit ```/etc/openstack-dashboard/local_settings.py```.
    ```
    OPENSTACK_HOST = "cap01"

    # Allow any host to access Dashboard.
    # Note, do not change the ```ALLOWED_HOSTS``` in Ubuntu configuration section
    ALLOWED_HOSTS = ['*']

    SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
    CACHES = {
        'default': {
             'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
             'LOCATION': 'cap01:11211',
        }
    }
    # Enable Identity API version 3
    OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

    # Enable support for domains
    OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

    # Configure API versions
    OPENSTACK_API_VERSIONS = {
        "identity": 3,
        "image": 2,
        "volume": 2,
    }

    # Configure the default domain
    OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

    # Configure the default role of users that you create through Dashboard
    OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

    # Enable networking services
    OPENSTACK_NEUTRON_NETWORK = {
        'enable_router': True,
        'enable_quotas': True,
        'enable_ipv6': True,
        'enable_distributed_router': False,
        'enable_ha_router': False,
        'enable_lb': True,
        'enable_firewall': True,
        'enable_vpn': True,
        'enable_fip_topology_check': True,
    }
    
    # Configure Timezone
    TIME_ZONE = "UTC"
    ```

3. Reload the web server configuration:
    ```
    # service apache2 reload
    ```

4. Error. If you cannot access through ```http://cap01.cs.purdue.edu/horizon/```, check ```/var/log/apache2/error.log```. If you found error like this ```IOError: [Errno 13] Permission denied: '/var/lib/openstack-dashboard/secret_key'```, change the owner and group of that file.
    ```
    $ ls -lht /var/lib/openstack-dashboard/
    total 12K
    drwxr-xr-x 10 www-data www-data 4.0K Jun 15 15:36 static
    -rw-------  1 root     root       64 Jun 15 15:36 secret_key
    -rw-r--r--  1 root     root        0 Jun 15 15:36 _var_lib_openstack-dashboard_secret_key.lock
    drwxr-xr-x  2 root     root     4.0K Apr  2 06:44 secret-key

    $ sudo chown -R www-data:www-data /var/lib/openstack-dashboard/secret_key
    ```

## <a name="exp"></a>Create Experiment Environment

* Create virtual networks
    1. Create provider network and subnet for external networking as administrator. This network is shared with other users.
        ```
        $ . admin-openrc
        $ openstack network create  --share --external \
            --provider-physical-network provider \
            --provider-network-type flat provider

        $ openstack subnet create --network provider \
            --allocation-pool start=128.10.130.204,end=128.10.130.210 \
            --dns-nameserver 128.10.2.5 --gateway 128.10.130.250 \
            --subnet-range 128.10.130.0/24 provider
        ```
    2. Create demo network and subnet as a regular user.
        ```
        $ . envi-openrc
        $ openstack network create demo-net
        $ openstack subnet create --network demo-net \
            --dns-nameserver 128.10.2.5 --gateway 192.168.1.1 \
            --subnet-range 192.168.1.0/24 demo-subnet
        ```
    3. Create a router and attach demo network to it and set gateway.
        ```
        $ openstack router create envi-router
        $ neutron router-interface-add envi-router demo-subnet
        $ neutron router-gateway-set envi-router provider
        ```
    4. Repeat step 2 and 3 to create ids network and subnet and attach it to the router.
    
* Launch instance with two NICs on demo network and ids network.
    1. Create image for ids instance.
        ```
        $ openstack image create "ids-1" --file hdd/images/ids-1-snapshot.raw --disk-format raw --container-format bare
        ```
    2. Create ports with fixed IPs on demo network and ids network.
        ```
        $ openstack port create --network demo-net --fixed-ip subnet=demo-subnet,ip-address=192.168.1.11 ids-1-demo-net
        $ openstack port create --network ids-net --fixed-ip subnet=ids-subnet,ip-address=192.168.2.11 ids-1-ids-net
        ```
    3. Launch instance with created image and ports.
        ```
        $ openstack server create --flavor m1.small --image ids-1 --security-group default \
          --nic port-id=91222c71-eb50-4e13-9e07-e5c5b8627e93 
          --nic port-id=99c19408-41a9-4052-88f6-0bcb32bf3eac 
          --availability-zone nova:wabash ids-1
        ```

## <a name="ref"></a>Reference
* Dell N series switch configuration manual, http://downloads.dell.com/Manuals/common/Networking_NxxUG_en-us.pdf
* Dell Networking N2000 Series Support, http://www.dell.com/support/home/us/en/04/product-support/product/networking-n2000-series/manuals
* Configuring VLAN on Dell Switches, http://www.dell.com/support/article/us/en/19/SLN294670/configuring-vlan-on-dell-switches?lang=EN
* OpenStack Ocata Installation Guide, https://docs.openstack.org/ocata/install-guide-ubuntu/
* OpenStack Kilo Installation Guide, https://www.linuxjust4u.com/wp-content/uploads/2017/01/openstack-install-guide-apt-kilo.pdf
* OpenStack Open vSwitch Mechanism Driver, https://docs.openstack.org/ocata/networking-guide/deploy-ovs.html
* OpenStack sfc-networking module, https://docs.openstack.org/developer/networking-sfc/
* Populate the Identity service database, https://bugs.launchpad.net/openstack-manuals/+bug/1612409
* Rescue broken Grub 2, https://help.ubuntu.com/community/Grub2/Installing#Reinstalling_GRUB_2
* Configuring VXLAN in Openstack Neutron, https://www.cloudenablers.com/blog/configuring-vxlan-in-openstack-neutron/
* OpenStack: Launching an instance on a specific compute host, http://www.googlinux.com/openstack-launching-an-instance-on-a-specific-compute-host/

