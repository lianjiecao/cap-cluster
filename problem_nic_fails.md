# Virtual NIC not sending packets

## What happens?
Virtual eth device stops sending pakcets after heavy transmission for a while. It does not happend to all virtual eth devices. But once it happens to a veth, it will happen again.

For example, ```eth2``` on ```ids-1``` stops sending packets after ~360 GB.

```bash
ubuntu@ids-1:~$ ifconfig
eth0      Link encap:Ethernet  HWaddr fa:16:3e:f1:79:7e
          inet addr:192.168.1.11  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fef1:797e/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:44319 errors:0 dropped:0 overruns:0 frame:0
          TX packets:41245 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:25771926 (25.7 MB)  TX bytes:15157396 (15.1 MB)

eth1      Link encap:Ethernet  HWaddr fa:16:3e:50:bd:23
          inet addr:192.168.2.11  Bcast:192.168.2.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fe50:bd23/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:880829925 errors:0 dropped:0 overruns:0 frame:0
          TX packets:860101026 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:938941592592 (938.9 GB)  TX bytes:916845091048 (916.8 GB)

eth2      Link encap:Ethernet  HWaddr fa:16:3e:48:9b:21
          inet addr:192.168.3.11  Bcast:192.168.3.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fe48:9b21/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:886849463 errors:0 dropped:0 overruns:0 frame:0
          TX packets:337128509 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:945372320709 (945.3 GB)  TX bytes:359370007989 (359.3 GB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:421 errors:0 dropped:0 overruns:0 frame:0
          TX packets:421 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:43800 (43.8 KB)  TX bytes:43800 (43.8 KB)
```

## Why?

It is difficult to pinpoint the exact problems. Based on the [**Linux networking stack**](https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/) and [**more details on the transmission path**](https://www.coverfire.com/articles/queueing-in-the-linux-network-stack/), everything other than the driver queue looks alright.

<img src="https://www.coverfire.com/wp-content/uploads/2012/11/figure_4_v2.png" width="480">

1. ``atop`` shows ``upper`` has equal number of RX and TX which mean it networking stack can receive and send packets correctly.
2. ``qdisc`` queue shows 1000 packets are backlogged, which means NIC driver is not getting packets from ``qdisc`` queue.

    ```bash
    ubuntu@ids-1:~$ tc -s qdisc show dev eth1
    qdisc pfifo_fast 0: root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
     Sent 916845100642 bytes 860101035 pkt (dropped 4724, overlimits 0 requeues 998)
     backlog 0b 0p requeues 998

    ubuntu@ids-1:~$ tc -s qdisc show dev eth2
    qdisc pfifo_fast 0: root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
     Sent 359370262763 bytes 337128748 pkt (dropped 523987, overlimits 0 requeues 537)
     backlog 1064934b 1000p requeues 537
    ```

It is very likely a problem of ``virtio`` driver. However, ``virtio`` is [**different from regular NIC drivers**](https://www.ibm.com/developerworks/library/l-virtio/index.html) (e.g., e1000, igb) as a paravirtualization solution. Therefore it is hard to say if it is a problem of the host or guest.

<img src="https://www.ibm.com/developerworks/library/l-virtio/figure1.gif">

## How to solve?

Similar problem is reported [**here**](https://bugs.centos.org/view.php?id=5526) using different distributions and kernel versions. 

Possible solutions:
- [x] Restart instance.  
	This can solve the problem completely.
- [x] Restart veth.  
	This does not solve the problem at all. If DHCP is used, it is even unable to renew DHCP lease.
- [x] Turn off packet segmentation/fragmentation options.  
	Edit ``/etc/rc.local`` with ``sudo ethtool --offload eth1 gso off tso off sg off gro off`` for all interfaces on affected guests and bridges on hosts.  
    But it does not seem to work in our cases.
- [x] Switch driver ``virtio`` to ``e1000`` in guests.  
	Doing this in OpenStack is a little complicated.  
    1. It can only be modified at image level. Update image with ``e1000`` driver for **all interfaces**.
    	```bash
        openstack@cap01:~$ glance image-update --property hw_vif_model=e1000 343140a1-4000-4617-bf9e-dec4e217bde1
        +------------------+--------------------------------------+
        | Property         | Value                                |
        +------------------+--------------------------------------+
        | base_image_ref   | c25ee9dc-431f-4dae-85e9-a5a26c7f4cbc |
        | boot_roles       | user                                 |
        | checksum         | 5c582a42cfeb546939d45296e77de2cf     |
        | container_format | bare                                 |
        | created_at       | 2017-10-05T00:01:33Z                 |
        | disk_format      | qcow2                                |
        | hw_vif_model     | e1000                                |
        | id               | 343140a1-4000-4617-bf9e-dec4e217bde1 |
        | image_location   | snapshot                             |
        | image_state      | available                            |
        | image_type       | snapshot                             |
        | instance_uuid    | 3596e534-a9ab-421f-8ea0-7ad0a149f670 |
        | min_disk         | 20                                   |
        | min_ram          | 0                                    |
        | name             | ids-1-before-suricata-4.0            |
        | owner            | 9b1a8bb7e17c492a9782a6678de94067     |
        | owner_id         | 9b1a8bb7e17c492a9782a6678de94067     |
        | protected        | False                                |
        | size             | 21159018496                          |
        | status           | active                               |
        | tags             | []                                   |
        | updated_at       | 2017-10-10T20:55:07Z                 |
        | user_id          | 9db1849f991b41cd86f1bcef118d4e7c     |
        | virtual_size     | None                                 |
        | visibility       | private                              |
        +------------------+--------------------------------------+
        ```
	2. Create new instance as usual and confirm the driver is switched to ``e1000``.
		```bash
        ubuntu@ids-1:~$ ethtool -i eth1
        driver: e1000
        version: 1.0.0
        firmware-version:
        bus-info: 0000:00:04.0
        supports-statistics: no
        supports-test: no
        supports-eeprom-access: no
        supports-register-dump: no
        supports-priv-flags: no
        ```
	However, this does not seem to work and, **more importantly**, the performance of e1000 driver on guest are very bad. It consumes a lot of CPU cycles to process packets. So, it is not a option for us.  
- [x] Consider this problem as a failure detection and recovery problem. 
	In this case, we use similar "hot swapping" method for workload generation instances. However, in order to avoid interrupting experiment as much as possible, we 
    1. Maintain two ids instances at the same time, one as "main" and the other as "backup".
    2. Run IDS and collect feature information on both of them. 
    3. But workload traffic is only redirected to the main instance and backup instance is "idle".
    4. When main instance fails, it delete the existing port chains and create new port chains to redirect traffic to the backup instance, swith roles of main and backup instance, and restart the old main instance and all process once back online.
    However, this does require extra steps in data processing to stitch data from the two instances and there will be interruption due to the process of deleting and creating port chains.
    
## Reference
* Monitoring and Tuning the Linux Networking Stack: Sending Data, https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/
* Traffic Control HOWTO, ftp://ftp.wayne.edu/ldp/en/Traffic-Control-HOWTO/Traffic-Control-HOWTO-single.html
* Queueing in the Linux Network Stack, http://www.linuxjournal.com/content/queueing-linux-network-stack?page=0,1
* 0005526: KVM Guest with virtio network loses network connectivity, https://bugs.centos.org/view.php?id=5526
* kvm virtio netdevs lose network connectivity under "enough" load, https://bugs.launchpad.net/ubuntu/+source/qemu-kvm/+bug/1325560
* Can't SSH between host and guest OS in VMware, http://www.tuxradar.com/answers/488
* Hey, make sure you disable tso and gso in your guest, http://blog.prgmr.com/xenophilia/2013/05/hey-make-sure-you-disable-tso.html
* Enabling OVS-DPDK in OpenStack, https://01.org/openstack/blogs/stephenfin/2016/enabling-ovs-dpdk-openstack
* 21.11. KVM Networking Performance, https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Administration_Guide/sect-Virtualization-Troubleshooting-KVM_networking_performance.html
* Using Open vSwitch with DPDK, https://github.com/openvswitch/ovs/blob/v2.5.0/INSTALL.DPDK.md
* Virtio: An I/O virtualization framework for Linux, https://www.ibm.com/developerworks/library/l-virtio/index.html
* Driver for VM Emulated Devices, http://dpdk.org/doc/guides/nics/e1000em.html
* Libvirt Custom Hardware Configuration, https://wiki.openstack.org/wiki/LibvirtCustomHardware

