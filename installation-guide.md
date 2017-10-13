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
* [Create Service Function Chaining](#sfc)
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
        sw-dell> enable
        sw-dell# config
        ```
    * Create VLAN with name ```openstack```:
        ```
        sw-dell(config)# vlan 100
        sw-dell(config-vlan100)# name openstack
        sw-dell(config-vlan100)# exit
        sw-dell(config)# vlan 100
        sw-dell(config-vlan100)# name openstack
        sw-dell(config-vlan100)# exit
        ```
        
    * Add port to VLAN (repeat to add all ports):
        ```
        sw-dell(config)#vlan 100
        sw-dell(config-vlan100)#name openstack
        sw-dell(config-vlan100)#exit
        ```

### <a name="host-net"></a>Host Networking

* Edit ```/etc/network/interfaces``` to configure NICs. Note, the second NIC is not necessary for controller in our setup.

**Controller node and compute nodes**
```
# External NIC for access and management
auto eth0
iface eth0 inet static
    address EXTERNAL_IP
    netmask 255.255.255.0
    network EXTERNAL_IP_NET
    broadcast EXTERNAL_IP_BR
    gateway EXTERNAL_GW
    dns-nameservers EXTERNAL_DNS
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
    address EXTERNAL_IP
    netmask 255.255.255.0
    network EXTERNAL_IP_NET
    broadcast EXTERNAL_IP_BR
    gateway EXTERNAL_GW
    # dns-* options are implemented by the resolvconf package, if installed
    dns-nameservers EXTERNAL_DNS
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
    EXTERNAL_IP_1    cap01    # controller node
    EXTERNAL_IP_2    cap02    # network node
    EXTERNAL_IP_3    wabash   # compute node
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

    * Click [<span style="text-decoration:underline">here</span>](https://raw.githubusercontent.com/lianjiecao/cap-cluster/master/ctl-neutron.conf) for a example configuration file.

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
In this example we create three network (demo-net, ids-net-1 and ids-net-2), one VNF instance (ids-1), seven workload instances (ids-snd-1, ids-snd-2, ids-snd-3, ids-snd-4, ids-snd-5, ids-rcv-1, and ids-rcv-2). 

|   Instances  | demo-net | ids-net-1 | ids-net-1 |
|:------------:|:--------:|:---------:|:---------:|
|     ids-1    |   eth0   |    eth1   |    eth2   |
|   ids-snd-1  |   eth0   |    eth1   |           |
|   ids-snd-2  |   eth0   |    eth1   |           |
|   ids-snd-3  |   eth0   |    eth1   |           |
|   ids-snd-4  |   eth0   |    eth2   |    eth2   |
|   ids-snd-5  |   eth0   |    eth2   |    eth2   |
|   ids-rcv-1  |   eth0   |    eth1   |           |
|   ids-rcv-2  |   eth0   |    eth1   |    eth2   |

demo-net is used as controll network.
Two SFC chains to create: 

1. [eth1@ids-snd-1,eth1@ids-snd-2,eth1@ids-snd-3] <=> [eth1@ids-1] <=> [eth1@ids-rcv-1,eth1@ids-rcv-2]
2. [eth2@ids-snd-4,eth2@ids-snd-5] <=> [eth2@ids-1] <=> [eth2@ids-rcv-2]

* Create virtual networks
    1. Create provider network and subnet for external networking as administrator. This network is shared with other users.
        ```
        $ . admin-openrc
        $ openstack network create  --share --external \
            --provider-physical-network provider \
            --provider-network-type flat provider

        $ openstack subnet create --network provider \
            --allocation-pool start=EXTERNAL_IP_START,end=EXTERNAL_IP_END \
            --dns-nameserver EXTERNAL_DNS --gateway EXTERNAL_GW \
            --subnet-range EXTERNAL_IP_NET provider
        ```
    2. Create demo network and subnet as a regular user.
        ```
        $ . envi-openrc
        $ openstack network create demo-net
        $ openstack subnet create --network demo-net \
            --dns-nameserver EXTERNAL_DNS --gateway 192.168.1.1 \
            --subnet-range 192.168.1.0/24 demo-subnet
        ```
    3. Create a router and attach demo network to it and set gateway.
        ```
        $ openstack router create envi-router
        $ neutron router-interface-add envi-router demo-subnet
        $ neutron router-gateway-set envi-router provider
        ```
    4. Repeat step 2 and 3 to create ids-net-1 and ids-net-2 network and subnet and attach them to the router.
        ```
        openstack@cap01:~$ openstack network list
        +--------------------------------------+-----------+--------------------------------------+
        | ID                                   | Name      | Subnets                              |
        +--------------------------------------+-----------+--------------------------------------+
        | 00052348-9c38-47f1-aa12-4caa4b13943e | ids-net-2 | afeb5442-a7a0-466b-828a-d50b2e39f624 |
        | 096021e0-b143-4a84-93c8-0641ca29affe | provider  | d3ff5ff5-d9b9-43da-a97d-0465b905f817 |
        | 33d92c2f-e2a7-43d2-a4d1-c2213bf8804b | ids-net-1 | e44a2bd1-d3d9-41b7-a434-a34d42386e8f |
        | 3b05f9e8-4ad1-4c2b-91b4-ead4828abdb9 | demo-net  | 5d77c479-eb1b-42ec-b990-15bd0f68fadb |
        +--------------------------------------+-----------+--------------------------------------+
        ```
    
* Launch instance with two NICs on demo network and ids network.
    1. Create image for ids instance.
        ```
        $ openstack image create "ids-1" --file hdd/images/ids-1-snapshot.raw --disk-format raw --container-format bare
        ```
    2. Create ports with fixed IPs on demo network and ids network for ids-1 and repeat for all instances.
        ```
        $ openstack port create --network demo-net --fixed-ip subnet=demo-subnet,ip-address=192.168.1.11 ids-1_demo-net
        $ openstack port create --network ids-net-1 --fixed-ip subnet=ids-subnet-1,ip-address=192.168.2.11 ids-1_ids-net-1
        $ openstack port create --network ids-net-2 --fixed-ip subnet=ids-subnet-2,ip-address=192.168.3.11 ids-1_ids-net-2
        ```
    3. Launch instance with created image and ports.
        ```
        openstack@cap01:~$ openstack server create --flavor m1.small --image ids-1 --security-group default \
            --nic port-id=06442390-e320-42d2-a0c5-29c8662e1b0c --nic port-id=f5484b38-0b14-4769-adb4-cd594cf1373c \
            --nic port-id=ea6e56cf-9a38-46db-891e-b12c09423e09 --key-name nfv-key --availability-zone nova:wabash ids-1
        ```

## <a name="sfc"></a>Create Service Function Chaining
This sections talks about how to create sfc using OpenStack module networking-sfc.
### On controller:
* Install networking-sfc module

    Install the corresponding version of networking-sfc:
    * Ocata: latest 4.0.x version
    * Newton: latest 3.0.x version
    * Mitaka: latest 2.0.x version

    Here we use 4.0 version:
    ```
    pip install -c https://git.openstack.org/cgit/openstack/requirements/plain/upper-constraints.txt?h=stable/ocata networking-sfc==4.0.0
    ```

* Configure Neutron and ML2 to use sfc module
    1. Edit ```/etc/neutron/neutron.conf``` to add ```service_plugin```, ```[ovs]``` and ```[flowclassifier]``` sections.
        ```
        [DEFAULT]
        ...
        service_plugins=router,networking_sfc.services.flowclassifier.plugin.FlowClassifierPlugin,networking_sfc.services.sfc.plugin.SfcPlugin
        [sfc]
        drivers = ovs
        [flowclassifier]
        drivers = ovs
        ```
    2. Restart neutron-server service.
        ```
        # service neutron-server restart
        ```
    3. Sync up database.
        ```
        # su -s /bin/sh -c "neutron-db-manage --subproject networking-sfc branches" neutron
        ```
    

* Create flow classifier, port pair, port pair groups and port chains that connects multiple VNF VMs.
    ```
    openstack@cap01:~$ openstack sfc flow classifier create --description "Traffic from ids-snd-1@ids-net-1 to ids-rcv-1@ids-net-1" \
        --ethertype IPv4 --source-ip-prefix 192.168.2.21/32 --destination-ip-prefix 192.168.2.31/32 \
        --logical-source-port ids-snd-1_ids-net-1 --logical-destination-port ids-rcv-1_ids-net-1 \
        fc_ids-snd-1_to_ids-rcv-1_ids-net-1
    +----------------------------+--------------------------------------+
    | Field                      | Value                                |
    +----------------------------+--------------------------------------+
    | description                | Traffic from ids-snd-1 to ids-rcv-1  |
    | destination_ip_prefix      | 192.168.2.31/32                      |
    | destination_port_range_max |                                      |
    | destination_port_range_min |                                      |
    | ethertype                  | IPv4                                 |
    | id                         | 6f75728c-5532-495d-8db9-a09866881b4e |
    | l7_parameters              | {}                                   |
    | logical_destination_port   | 53ef0cf4-a640-4a03-af65-2f21a51ad18b |
    | logical_source_port        | 6bad8ea1-87c8-4157-b7cf-ca245113d343 |
    | name                       | fc_ids-snd-1_to_ids-rcv-1            |
    | project_id                 | 9b1a8bb7e17c492a9782a6678de94067     |
    | protocol                   |                                      |
    | source_ip_prefix           | 192.168.2.21/32                      |
    | source_port_range_max      |                                      |
    | source_port_range_min      |                                      |
    | tenant_id                  | 9b1a8bb7e17c492a9782a6678de94067     |
    +----------------------------+--------------------------------------+

    openstack@cap01:~$ openstack sfc flow classifier create --description "Traffic from ids-rcv-1@ids-net-1 to ids-snd-1@ids-net-1" \
        --ethertype IPv4 --source-ip-prefix 192.168.2.31/32 --destination-ip-prefix 192.168.2.21/32 \
        --logical-source-port ids-rcv-1-ids-net --logical-destination-port ids-snd-1-ids-net \
        fc_ids-rcv-1_to_ids-snd-1_ids-net-1
    +----------------------------+--------------------------------------+
    | Field                      | Value                                |
    +----------------------------+--------------------------------------+
    | description                | Traffic from ids-rcv-1 to ids-snd-1  |
    | destination_ip_prefix      | 192.168.2.21/32                      |
    | destination_port_range_max |                                      |
    | destination_port_range_min |                                      |
    | ethertype                  | IPv4                                 |
    | id                         | 3f80c0cb-ef0d-4572-8cca-452ddb7926ec |
    | l7_parameters              | {}                                   |
    | logical_destination_port   | 6bad8ea1-87c8-4157-b7cf-ca245113d343 |
    | logical_source_port        | 53ef0cf4-a640-4a03-af65-2f21a51ad18b |
    | name                       | fc_ids-rcv-1_to_ids-snd-1            |
    | project_id                 | 9b1a8bb7e17c492a9782a6678de94067     |
    | protocol                   |                                      |
    | source_ip_prefix           | 192.168.2.31/32                      |
    | source_port_range_max      |                                      |
    | source_port_range_min      |                                      |
    | tenant_id                  | 9b1a8bb7e17c492a9782a6678de94067     |
    +----------------------------+--------------------------------------+
    
    [Repeat to create more flow classifiers ...]

    openstack@cap01:~/cap-cluster$ openstack sfc flow classifier list
    +--------------------------------------+-------------------------------------+----------+-----------------+-----------------+--------------------------------------+--------------------------------------+
    | ID                                   | Name                                | Protocol | Source-IP       | Destination-IP  | Logical-Source-Port                  | Logical-Destination-Port             |
    +--------------------------------------+-------------------------------------+----------+-----------------+-----------------+--------------------------------------+--------------------------------------+
    | 1ef1f76f-46ca-43ea-80c3-84b961368d1f | fc_ids-snd-3_to_ids-rcv-2_ids-net-1 | None     | 192.168.2.23/32 | 192.168.2.32/32 | e0dac140-abcb-47d7-8945-2eeb1ec036ff | e06e403c-0fbb-457c-87a4-02030c5df5d4 |
    | 3f80c0cb-ef0d-4572-8cca-452ddb7926ec | fc_ids-rcv-1_to_ids-snd-1_ids-net-1 | None     | 192.168.2.31/32 | 192.168.2.21/32 | 53ef0cf4-a640-4a03-af65-2f21a51ad18b | 6bad8ea1-87c8-4157-b7cf-ca245113d343 |
    | 6f75728c-5532-495d-8db9-a09866881b4e | fc_ids-snd-1_to_ids-rcv-1_ids-net-1 | None     | 192.168.2.21/32 | 192.168.2.31/32 | 6bad8ea1-87c8-4157-b7cf-ca245113d343 | 53ef0cf4-a640-4a03-af65-2f21a51ad18b |
    | 98801a52-dcff-4d7c-816b-96567981bdff | fc_ids-rcv-2_to_ids-snd-5_ids-net-2 | None     | 192.168.3.32/32 | 192.168.3.25/32 | 9f248c8a-c675-426c-b3aa-14f3acfc011f | 911f4d96-3f69-46cb-9bf1-ab55ed8dba76 |
    | a454e7ed-1885-4a93-af86-0ed31c0bf915 | fc_ids-snd-2_to_ids-rcv-2_ids-net-1 | None     | 192.168.2.22/32 | 192.168.2.32/32 | 427eb3c6-98f9-43e9-bde0-20b780e6783f | e06e403c-0fbb-457c-87a4-02030c5df5d4 |
    | d8bf0d8d-1788-445e-9606-9aaca6b74c9e | fc_ids-rcv-2_to_ids-snd-4_ids-net-2 | None     | 192.168.3.32/32 | 192.168.3.24/32 | 9f248c8a-c675-426c-b3aa-14f3acfc011f | fbbc7cf2-00f4-410d-b0f2-b255735e29c6 |
    | d8e86849-6d94-4b64-9a1e-6178563d5e96 | fc_ids-rcv-2_to_ids-snd-2_ids-net-1 | None     | 192.168.2.32/32 | 192.168.2.22/32 | e06e403c-0fbb-457c-87a4-02030c5df5d4 | 427eb3c6-98f9-43e9-bde0-20b780e6783f |
    | e50d8c54-cbcc-47b2-9a98-7e97155b53bb | fc_ids-snd-5_to_ids-rcv-2_ids-net-2 | None     | 192.168.3.25/32 | 192.168.3.32/32 | 911f4d96-3f69-46cb-9bf1-ab55ed8dba76 | 9f248c8a-c675-426c-b3aa-14f3acfc011f |
    | f44c89d3-0886-45c3-94ab-91b380cebef5 | fc_ids-snd-4_to_ids-rcv-2_ids-net-2 | None     | 192.168.3.24/32 | 192.168.3.32/32 | fbbc7cf2-00f4-410d-b0f2-b255735e29c6 | 9f248c8a-c675-426c-b3aa-14f3acfc011f |
    | f4e2f64e-09de-44f6-ae5d-9137bc62727d | fc_ids-rcv-2_to_ids-snd-3_ids-net-1 | None     | 192.168.2.32/32 | 192.168.2.23/32 | e06e403c-0fbb-457c-87a4-02030c5df5d4 | e0dac140-abcb-47d7-8945-2eeb1ec036ff |
    +--------------------------------------+-------------------------------------+----------+-----------------+-----------------+--------------------------------------+--------------------------------------+

    openstack@cap01:~$ openstack sfc port pair create --description "ids-1@ids-net-1" --ingress ids-1_ids-net-1 --egress ids-1_ids-net-1 pp_ids-1_ids-net-1
    +-----------------------------+--------------------------------------+
    | Field                       | Value                                |
    +-----------------------------+--------------------------------------+
    | description                 | ids-1@ids-net-1                      |
    | egress                      | 2d2eae7a-861c-4215-a77a-fd0abf8a19a1 |
    | id                          | 2ed66e91-326c-4fda-80da-0498caa2d298 |
    | ingress                     | 2d2eae7a-861c-4215-a77a-fd0abf8a19a1 |
    | name                        | pp_ids-2_ids-net-1                   |
    | project_id                  | 9b1a8bb7e17c492a9782a6678de94067     |
    | service_function_parameters | {u'weight': 1, u'correlation': None} |
    +-----------------------------+--------------------------------------+

    openstack@cap01:~$ openstack sfc port pair create --description "ids-1@ids-net-2" --ingress ids-1_ids-net-2 --egress ids-1_ids-net-2 pp_ids-1_ids-net-2
    +-----------------------------+--------------------------------------+
    | Field                       | Value                                |
    +-----------------------------+--------------------------------------+
    | description                 | ids-1@ids-net-2                      |
    | egress                      | edaa470b-bbbd-47e7-9f52-388fac91f336 |
    | id                          | 5cd7c5a3-09eb-4bc9-a28e-4568a5c92bf2 |
    | ingress                     | edaa470b-bbbd-47e7-9f52-388fac91f336 |
    | name                        | pp_ids-2_ids-net-2                   |
    | project_id                  | 9b1a8bb7e17c492a9782a6678de94067     |
    | service_function_parameters | {u'weight': 1, u'correlation': None} |
    +-----------------------------+--------------------------------------+

    [Repeat to create port pairs for all VNF instances ...]

    openstack@cap01:~$ openstack sfc port pair list
    +--------------------------------------+--------------------+--------------------------------------+--------------------------------------+
    | ID                                   | Name               | Ingress Logical Port                 | Egress Logical Port                  |
    +--------------------------------------+--------------------+--------------------------------------+--------------------------------------+
    | 17809479-dd8e-4de3-91dd-2efb60310719 | pp_ids-1_ids-net-1 | f5484b38-0b14-4769-adb4-cd594cf1373c | f5484b38-0b14-4769-adb4-cd594cf1373c |
    | 2c8e2ed8-ac9c-4ca2-a9fd-6e5a40916f06 | pp_ids-1_ids-net-2 | ea6e56cf-9a38-46db-891e-b12c09423e09 | ea6e56cf-9a38-46db-891e-b12c09423e09 |
    | 2ed66e91-326c-4fda-80da-0498caa2d298 | pp_ids-2_ids-net-1 | 2d2eae7a-861c-4215-a77a-fd0abf8a19a1 | 2d2eae7a-861c-4215-a77a-fd0abf8a19a1 |
    | 5cd7c5a3-09eb-4bc9-a28e-4568a5c92bf2 | pp_ids-2_ids-net-2 | edaa470b-bbbd-47e7-9f52-388fac91f336 | edaa470b-bbbd-47e7-9f52-388fac91f336 |
    +--------------------------------------+--------------------+--------------------------------------+--------------------------------------+

    openstack@cap01:~$ openstack sfc port pair group create --port-pair pp_ids-1_ids-net-1 ppg_ids-1_ids-net-1
    +----------------------------+-------------------------------------------+
    | Field                      | Value                                     |
    +----------------------------+-------------------------------------------+
    | description                |                                           |
    | group_id                   | 3                                         |
    | id                         | 36cc933f-4dcd-4ffb-b0a8-43034e767d4b      |
    | name                       | ppg_ids-1_ids-net-1                       |
    | port_pair_group_parameters | {u'lb_fields': []}                        |
    | port_pairs                 | [u'2ed66e91-326c-4fda-80da-0498caa2d298'] |
    | project_id                 | 9b1a8bb7e17c492a9782a6678de94067          |
    +----------------------------+-------------------------------------------+
    openstack@cap01:~$ openstack sfc port pair group create --port-pair pp_ids-1_ids-net-2 ppg_ids-1_ids-net-2
    +----------------------------+-------------------------------------------+
    | Field                      | Value                                     |
    +----------------------------+-------------------------------------------+
    | description                |                                           |
    | group_id                   | 4                                         |
    | id                         | 39409ac7-5ec8-4cec-a75a-cdbef698e6b7      |
    | name                       | ppg_ids-1_ids-net-2                       |
    | port_pair_group_parameters | {u'lb_fields': []}                        |
    | port_pairs                 | [u'5cd7c5a3-09eb-4bc9-a28e-4568a5c92bf2'] |
    | project_id                 | 9b1a8bb7e17c492a9782a6678de94067          |
    +----------------------------+-------------------------------------------+

    [Repeat to create port pair groups for all VNF instances ...]

    openstack@cap01:~$ openstack sfc port pair group list
    +--------------------------------------+---------------------+-------------------------------------------+----------------------------+
    | ID                                   | Name                | Port Pair                                 | Port Pair Group Parameters |
    +--------------------------------------+---------------------+-------------------------------------------+----------------------------+
    | 0eab9ebd-b523-4278-b378-09ad553c0b91 | ppg_ids-1_ids-net-1 | [u'17809479-dd8e-4de3-91dd-2efb60310719'] | {u'lb_fields': []}         |
    | 36cc933f-4dcd-4ffb-b0a8-43034e767d4b | ppg_ids-2_ids-net-1 | [u'2ed66e91-326c-4fda-80da-0498caa2d298'] | {u'lb_fields': []}         |
    | 39409ac7-5ec8-4cec-a75a-cdbef698e6b7 | ppg_ids-2_ids-net-2 | [u'5cd7c5a3-09eb-4bc9-a28e-4568a5c92bf2'] | {u'lb_fields': []}         |
    | d5f72f5c-537a-4c28-a9fc-f9ed86412f8d | ppg_ids-1_ids-net-2 | [u'2c8e2ed8-ac9c-4ca2-a9fd-6e5a40916f06'] | {u'lb_fields': []}         |
    +--------------------------------------+---------------------+-------------------------------------------+----------------------------+


    openstack@cap01:~$ openstack sfc port chain create --port-pair-group ppg_ids-1_ids-net-1 --flow-classifier fc_ids-snd-1_to_ids-rcv-1_ids-net-1 \
        --flow-classifier fc_ids-rcv-1_to_ids-snd-1_ids-net-1 --flow-classifier fc_ids-snd-2_to_ids-rcv-2_ids-net-1 \
        --flow-classifier fc_ids-rcv-2_to_ids-snd-2_ids-net-1 --flow-classifier fc_ids-snd-3_to_ids-rcv-2_ids-net-1 \
        --flow-classifier fc_ids-rcv-2_to_ids-snd-3_ids-net-1 pc_ids-1_ids-net-1
    +------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | Field            | Value                                                                                                                                                                    |
    +------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | chain_id         | 1                                                                                                                                                                        |
    | chain_parameters | {u'symmetric': False, u'correlation': u'mpls'}                                                                                                                           |
    | description      |                                                                                                                                                                          |
    | flow_classifiers | [u'6f75728c-5532-495d-8db9-a09866881b4e', u'3f80c0cb-ef0d-4572-8cca-452ddb7926ec', u'a454e7ed-1885-4a93-af86-0ed31c0bf915', u'd8e86849-6d94-4b64-9a1e-6178563d5e96',     |
    |                  | u'1ef1f76f-46ca-43ea-80c3-84b961368d1f', u'f4e2f64e-09de-44f6-ae5d-9137bc62727d']                                                                                        |
    | id               | d8d9eaf5-6c96-4f56-a2fc-a0ba06db2bdc                                                                                                                                     |
    | name             | pc_ids-1_ids-net-1                                                                                                                                                       |
    | port_pair_groups | [u'0eab9ebd-b523-4278-b378-09ad553c0b91']                                                                                                                                |
    | project_id       | 9b1a8bb7e17c492a9782a6678de94067                                                                                                                                         |
    +------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

    openstack@cap01:~$ openstack sfc port chain create --port-pair-group ppg_ids-1_ids-net-2 --flow-classifier fc_ids-snd-4_to_ids-rcv-2_ids-net-2 \
        --flow-classifier fc_ids-rcv-2_to_ids-snd-4_ids-net-2 --flow-classifier fc_ids-snd-5_to_ids-rcv-2_ids-net-2 \
        --flow-classifier fc_ids-rcv-2_to_ids-snd-5_ids-net-2 pc_ids-1_ids-net-2
    +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | Field            | Value                                                                                                                                                                |
    +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | chain_id         | 2                                                                                                                                                                    |
    | chain_parameters | {u'symmetric': False, u'correlation': u'mpls'}                                                                                                                       |
    | description      |                                                                                                                                                                      |
    | flow_classifiers | [u'f44c89d3-0886-45c3-94ab-91b380cebef5', u'd8bf0d8d-1788-445e-9606-9aaca6b74c9e', u'e50d8c54-cbcc-47b2-9a98-7e97155b53bb', u'98801a52-dcff-4d7c-816b-96567981bdff'] |
    | id               | 4eb04f61-a1e2-4342-94cc-08251f62d04a                                                                                                                                 |
    | name             | pc_ids-1_ids-net-2                                                                                                                                                   |
    | port_pair_groups | [u'd5f72f5c-537a-4c28-a9fc-f9ed86412f8d']                                                                                                                            |
    | project_id       | 9b1a8bb7e17c492a9782a6678de94067                                                                                                                                     |
    +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    openstack@cap01:~$ openstack sfc port chain list
    +--------------------------------------+--------------------+-------------------------------------------+---------------------------------------------------------------------------------------+------------------------------------------------+
    | ID                                   | Name               | Port Pair Groups                          | Flow Classifiers                                                                      | Chain Parameters                               |
    +--------------------------------------+--------------------+-------------------------------------------+---------------------------------------------------------------------------------------+------------------------------------------------+
    | 4eb04f61-a1e2-4342-94cc-08251f62d04a | pc_ids-1_ids-net-2 | [u'd5f72f5c-537a-4c28-a9fc-f9ed86412f8d'] | [u'98801a52-dcff-4d7c-816b-96567981bdff', u'd8bf0d8d-1788-445e-9606-9aaca6b74c9e',    | {u'symmetric': False, u'correlation': u'mpls'} |
    |                                      |                    |                                           | u'e50d8c54-cbcc-47b2-9a98-7e97155b53bb', u'f44c89d3-0886-45c3-94ab-91b380cebef5']     |                                                |
    | d8d9eaf5-6c96-4f56-a2fc-a0ba06db2bdc | pc_ids-1_ids-net-1 | [u'0eab9ebd-b523-4278-b378-09ad553c0b91'] | [u'1ef1f76f-46ca-43ea-80c3-84b961368d1f', u'3f80c0cb-ef0d-4572-8cca-452ddb7926ec',    | {u'symmetric': False, u'correlation': u'mpls'} |
    |                                      |                    |                                           | u'6f75728c-5532-495d-8db9-a09866881b4e', u'a454e7ed-1885-4a93-af86-0ed31c0bf915',     |                                                |
    |                                      |                    |                                           | u'd8e86849-6d94-4b64-9a1e-6178563d5e96', u'f4e2f64e-09de-44f6-ae5d-9137bc62727d']     |                                                |
    +--------------------------------------+--------------------+-------------------------------------------+---------------------------------------------------------------------------------------+------------------------------------------------+


    ```

### On compute node:
* Install networking-sfc module

    Repeat the same process to install ```networking-sfc```.
    ```
    pip install -c https://git.openstack.org/cgit/openstack/requirements/plain/upper-constraints.txt?h=stable/ocata networking-sfc==4.0.0
    ```

* Configure neutron-openvswitch agent to use sfc extension
    1. Edit ```/etc/neutron/plugins/ml2/openvswitch_agent.ini``` to add ```extensions```.
        ```
        [agent]
        ...
        extensions = sfc
        ```
    2. Restart openvswitch agent.
        ```
        # service neutron-openvswitch-agent restart
    ```

* Modify iptables rules to allow outgoing traffic on VNF VMs
    ```
    openstack@wabash:~$ sudo iptables -L | grep 192.168.2.11 -C 2
    Chain neutron-openvswi-sf5484b38-0 (1 references)
    target     prot opt source               destination
    RETURN     all  --  192.168.2.11         anywhere             MAC FA:16:3E:47:8B:F4 /* Allow traffic from defined IP/MAC pairs. \*/
    DROP       all  --  anywhere             anywhere


    openstack@wabash:~$ sudo iptables -D neutron-openvswi-sf5484b38-0 2; sudo iptables -A neutron-openvswi-sf5484b38-0 -j RETURN
    openstack@wabash:~$ sudo iptables -L | grep 192.168.2.11 -C 2
    Chain neutron-openvswi-sf5484b38-0 (1 references)
    target     prot opt source               destination
    RETURN     all  --  192.168.2.11         anywhere             MAC FA:16:3E:47:8B:F4 /* Allow traffic from defined IP/MAC pairs. */
    RETURN     all  --  anywhere             anywhere


    
    openstack@wabash:~$ sudo iptables -L | grep 192.168.3.11 -C 2
    Chain neutron-openvswi-sea6e56cf-9 (1 references)
    target     prot opt source               destination
    RETURN     all  --  192.168.3.11         anywhere             MAC FA:16:3E:CD:AB:79 /* Allow traffic from defined IP/MAC pairs. */
    DROP       all  --  anywhere             anywhere

    
    openstack@wabash:~$ sudo iptables -D neutron-openvswi-sea6e56cf-9 2; sudo iptables -A neutron-openvswi-sea6e56cf-9 -j RETURN
    openstack@wabash:~$ sudo iptables -L | grep 192.168.2.11 -C 2
    Chain neutron-openvswi-sea6e56cf-9 (1 references)
    target     prot opt source               destination
    RETURN     all  --  192.168.3.11         anywhere             MAC FA:16:3E:CD:AB:79 /* Allow traffic from defined IP/MAC pairs. */
    RETURN     all  --  anywhere             anywhere

    ```

### On IDS VM:
This section shows how to configure Suricata in inline mode.
Traffic comes in from eth2 (neutron port ids-1-ids-net-1) and redirected to nfqueue 0 by iptables for Suricata to inspect.

* Enable IP forwarding on the fly
    ```
    ubuntu@ids-1:~$ sudo sysctl -w net.ipv4.ip_forward=1
    net.ipv4.ip_forward = 1
    ```
    or permanently by editing ```/etc/sysctl.conf```
    ```
    net.ipv4.ip_forward = 1
    $ sudo sysctl -p /etc/sysctl.conf
    ```

* Check routing infomation
    ```
    ubuntu@ids-1:~$ route -n
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    0.0.0.0         192.168.2.1     0.0.0.0         UG    0      0        0 eth1
    169.254.169.254 192.168.2.1     255.255.255.255 UGH   0      0        0 eth1
    192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
    192.168.2.0     0.0.0.0         255.255.255.0   U     0      0        0 eth1
    192.168.3.0     0.0.0.0         255.255.255.0   U     0      0        0 eth2
    ```

* Configure iptables to use nfqueue for forward traffic
    ```
    ubuntu@ids-1:~$ sudo iptables -I FORWARD -i eth1 -j NFQUEUE --queue-num 0
    ubuntu@ids-1:~$ sudo iptables -I FORWARD -i eth2 -j NFQUEUE --queue-num 1
    ubuntu@ids-1:~$ sudo iptables -vnL
    Chain INPUT (policy ACCEPT 9 packets, 660 bytes)
     pkts bytes target     prot opt in     out     source               destination

    Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
     pkts bytes target     prot opt in     out     source               destination
        0     0 NFQUEUE    all  --  eth2   *       0.0.0.0/0            0.0.0.0/0            NFQUEUE num 1
        0     0 NFQUEUE    all  --  eth1   *       0.0.0.0/0            0.0.0.0/0            NFQUEUE num 0

    Chain OUTPUT (policy ACCEPT 5 packets, 728 bytes)
     pkts bytes target     prot opt in     out     source               destination

     [ping receiving VM from sending VM and confirm ICMP packets are redirected to VNF instances ...]

    ubuntu@ids-1:~$ sudo iptables -vnL
    Chain INPUT (policy ACCEPT 301 packets, 21588 bytes)
     pkts bytes target     prot opt in     out     source               destination

    Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
     pkts bytes target     prot opt in     out     source               destination
        3   252 NFQUEUE    all  --  eth2   *       0.0.0.0/0            0.0.0.0/0            NFQUEUE num 1
        3   252 NFQUEUE    all  --  eth1   *       0.0.0.0/0            0.0.0.0/0            NFQUEUE num 0

    Chain OUTPUT (policy ACCEPT 196 packets, 30540 bytes)
     pkts bytes target     prot opt in     out     source               destination

    $ sudo apt install iptables-persistent
    ```

* Run Suricata with inline mode
    ```
    ubuntu@ids-1:~$ sudo suricata -c /etc/suricata/suricata.yaml -q 0 -q 1 -vv
    29/6/2017 -- 01:33:26 - <Notice> - This is Suricata version 3.2.1 RELEASE
    29/6/2017 -- 01:33:26 - <Info> - CPUs/cores online: 1
    29/6/2017 -- 01:33:26 - <Info> - NFQ running in standard ACCEPT/DROP mode
    29/6/2017 -- 01:33:26 - <Info> - Running in live mode, activating unix socket
    29/6/2017 -- 01:33:30 - <Info> - 45 rule files processed. 19602 rules successfully loaded, 0 rules failed
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for tcp-packet
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for tcp-stream
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for udp-packet
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for other-ip
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_uri
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_request_line
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_client_body
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_response_line
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_header
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_header
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_raw_header
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_raw_header
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_method
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_cookie
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_cookie
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_raw_uri
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_user_agent
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_host
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_raw_host
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_stat_msg
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_stat_code
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for dns_query
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for tls_sni
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for tls_cert_issuer
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for tls_cert_subject
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for file_data
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for file_data
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_request_line
    29/6/2017 -- 01:33:30 - <Perf> - using shared mpm ctx' for http_response_line
    29/6/2017 -- 01:33:30 - <Info> - 19610 signatures processed. 1269 are IP-only rules, 6635 are inspecting packet payload, 14191 inspect application layer, 0 are decoder event only
    29/6/2017 -- 01:33:30 - <Perf> - TCP toserver: 41 port groups, 34 unique SGH's, 7 copies
    29/6/2017 -- 01:33:30 - <Perf> - TCP toclient: 21 port groups, 21 unique SGH's, 0 copies
    29/6/2017 -- 01:33:30 - <Perf> - UDP toserver: 41 port groups, 30 unique SGH's, 11 copies
    29/6/2017 -- 01:33:30 - <Perf> - UDP toclient: 21 port groups, 16 unique SGH's, 5 copies
    29/6/2017 -- 01:33:30 - <Perf> - OTHER toserver: 254 proto groups, 3 unique SGH's, 251 copies
    29/6/2017 -- 01:33:30 - <Perf> - OTHER toclient: 254 proto groups, 0 unique SGH's, 254 copies
    29/6/2017 -- 01:33:31 - <Perf> - Unique rule groups: 104
    29/6/2017 -- 01:33:31 - <Perf> - Builtin MPM "toserver TCP packet": 25
    29/6/2017 -- 01:33:31 - <Perf> - Builtin MPM "toclient TCP packet": 20
    29/6/2017 -- 01:33:31 - <Perf> - Builtin MPM "toserver TCP stream": 24
    29/6/2017 -- 01:33:31 - <Perf> - Builtin MPM "toclient TCP stream": 21
    29/6/2017 -- 01:33:31 - <Perf> - Builtin MPM "toserver UDP packet": 30
    29/6/2017 -- 01:33:31 - <Perf> - Builtin MPM "toclient UDP packet": 15
    29/6/2017 -- 01:33:31 - <Perf> - Builtin MPM "other IP packet": 2
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toserver http_uri": 8
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toserver http_client_body": 6
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toserver http_header": 7
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toclient http_header": 3
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toserver http_raw_header": 1
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toclient http_raw_header": 1
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toserver http_method": 4
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toserver http_cookie": 1
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toclient http_cookie": 2
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toserver http_raw_uri": 2
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toserver http_user_agent": 3
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toclient http_stat_code": 1
    29/6/2017 -- 01:33:31 - <Perf> - AppLayer MPM "toclient file_data": 5
    29/6/2017 -- 01:33:32 - <Info> - Threshold config parsed: 0 rule(s) found
    29/6/2017 -- 01:33:32 - <Info> - eve-log output device (regular) initialized: eve.json
    29/6/2017 -- 01:33:32 - <Info> - http-log output device (regular) initialized: http.log
    29/6/2017 -- 01:33:32 - <Info> - tls-log output device (regular) initialized: tls.log
    29/6/2017 -- 01:33:32 - <Info> - dns-log output device (regular) initialized: dns.log
    29/6/2017 -- 01:33:32 - <Info> - dns-log output device (regular) initialized: dns.log
    29/6/2017 -- 01:33:32 - <Info> - stats output device (regular) initialized: stats.log
    29/6/2017 -- 01:33:32 - <Info> - binding this thread 0 to queue '0'
    29/6/2017 -- 01:33:32 - <Info> - setting queue length to 4096
    29/6/2017 -- 01:33:32 - <Info> - setting nfnl bufsize to 6144000
    29/6/2017 -- 01:33:32 - <Info> - Running in live mode, activating unix socket
    29/6/2017 -- 01:33:32 - <Info> - Using unix socket file '/var/run/suricata/suricata-command.socket'
    29/6/2017 -- 01:33:32 - <Info> - Created socket directory /var/run/suricata/
    29/6/2017 -- 01:33:32 - <Notice> - all 3 packet processing threads, 4 management threads initialized, engine started.
    ^C29/6/2017 -- 01:33:49 - <Notice> - Signal Received.  Stopping engine.
    29/6/2017 -- 01:33:49 - <Perf> - 0 new flows, 0 established flows were timed out, 0 flows in closed state
    29/6/2017 -- 01:33:50 - <Info> - time elapsed 17.996s
    29/6/2017 -- 01:33:51 - <Perf> - 0 flows processed
    29/6/2017 -- 01:33:51 - <Notice> - (RX-Q0) Treated: Pkts 20, Bytes 1680, Errors 0
    29/6/2017 -- 01:33:51 - <Notice> - (RX-Q0) Verdict: Accepted 20, Dropped 0, Replaced 0
    29/6/2017 -- 01:33:51 - <Perf> - AutoFP - Total flow handler queues - 1
    29/6/2017 -- 01:33:51 - <Info> - TLS logger logged 0 requests
    29/6/2017 -- 01:33:51 - <Info> - DNS logger logged 0 transactions
    29/6/2017 -- 01:33:51 - <Info> - DNS logger logged 0 transactions
    29/6/2017 -- 01:33:51 - <Perf> - ippair memory usage: 398144 bytes, maximum: 16777216
    29/6/2017 -- 01:33:51 - <Perf> - host memory usage: 398144 bytes, maximum: 33554432
    29/6/2017 -- 01:33:51 - <Info> - cleaning up signature grouping structure... complete
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
* networking-sfc API 5.0, https://docs.openstack.org/networking-sfc/latest/contributor/api.html
* networking-sfc commands, https://docs.openstack.org/python-neutronclient/latest/cli/osc/v2/networking-sfc.html
* networking-sfc documentation, https://docs.openstack.org/developer/networking-sfc/
* networking-sfc developer document, https://docs.openstack.org/developer/networking-sfc/
* networking-sfc configuration, https://docs.openstack.org/newton/networking-guide/config-sfc.html
* networking-sfc source code, https://github.com/openstack/networking-sfc
* How to Enable IP Forwarding in Linux, http://www.ducea.com/2006/08/01/how-to-enable-ip-forwarding-in-linux/
* Changing IP Addresses and Routes, http://linux-ip.net/html/basic-changing.html
* Firewall and NAT service in Linux, http://cn.linux.vbird.org/linux_server/0250simple_firewall.php#nat_what
* iptables FORWARD and INPUT, https://stackoverflow.com/questions/12945233/iptables-forward-and-input
* Iptables: Forwarding packets doesn't work, https://serverfault.com/questions/385251/iptables-forwarding-packets-doesnt-work
* Setting up IPS/inline for Linux, http://suricata.readthedocs.io/en/latest/setting-up-ipsinline-for-linux.html
* Snort IPS Inline Mode on Ubuntu, http://sublimerobots.com/2016/02/snort-ips-inline-mode-on-ubuntu/

