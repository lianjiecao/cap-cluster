# Traffic Generation, Capture and Monitoring

## Traffic Traces
### SimpleWeb
[**SimpleWeb**](https://www.simpleweb.org/wiki/index.php/Traces) includes multiple sets of traffic traces for different purposes (e.g., DNS, DDoS, DNSSEC, SSH, Cloud storage, campus network, NetFlow, etc).

### CAIDA
[**CAIDA**](http://www.caida.org/data/overview/) has a lot of traces include AS topologies and Internet traces (2008~2016). But most of them require permissions to access.

### Others
* **LBNL/ICSI Enterprise Tracing Project** offers a set of 11GB of packet header traces from October 2004 through January 2005 is available via FTP.  
	http://www.icir.org/enterprise-tracing/download.html
	
## Tools
### Generation
* **Iperf** and **Iperf3**, one of the most famous traffic generation tool.  
	https://iperf.fr/iperf-download.php
* **hping3** is a command-line oriented TCP/IP packet assembler/analyzer.  
	http://www.hping.org/
* **MoonGen** is a fully scriptable high-speed packet generator built on DPDK and LuaJIT.  
	https://github.com/emmericp/MoonGen
* **Pktgen** is a traffic generator powered by Intel's DPDK.  
	https://github.com/pktgen/Pktgen-DPDK
* **D-ITG** is a platform capable to produce traffic at packet level accurately replicating appropriate stochastic processes for both IDT (Inter Departure Time) and PS (Packet Size) random variables.  
	http://traffic.comics.unina.it/software/ITG/
    
* **Tcpreplay** is a suite of free Open Source utilities for editing and replaying previously captured network	traffic.  
	http://tcpreplay.appneta.com/
* **Harpoon** is a flow-level traffic generator based a set of distributional parameters that can be automatically extracted from Netflow traces.
	http://cs.colgate.edu/~jsommers/harpoon/

### Monitoring
Refer to [**Stanford SLAC**](http://www.slac.stanford.edu/xorg/nmtf/nmtf-tools.html) for a complete network monitoring tools.  
[**CAIDA**](http://www.caida.org/tools/) also lists a number of comprehensive networking related tools.

#### Other Usefull Links
* PktAnonpacket trace anonymization tools.  
 	https://github.com/caesar0301/awesome-pcaptools
* Network trace processing tools.  
	https://github.com/caesar0301/awesome-pcaptools
* A Survey of Network Traffic Monitoring and Analysis Tools.  
	http://www.cs.wustl.edu/~jain/cse567-06/ftp/net_traffic_monitors3/#1


## Evaluation of Other Publications

### E2: A Framework for NFV Applications

Dynamic Scaling. We drive the NF with 1 Gbps of input traffic, with 2,000 new flows arriving each second on average and flow length distributions drawn from published measurement studies [32]. This results in a total load of approximately 10,000 active concurrent flows and hence dynamic scaling (effectively) requires 'shifting' load equivalent to 5,000 flows off the original NF.

[32] LEE, D., AND BROWNLEE, N. Passive Measurement of One-way and Two-way Flow Lifetimes. ACM SIGCOMM Computer Communications Review 37, 3 (November 2007).  
http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.649.4724&rep=rep1&type=pdf


### OpenNF: Enabling Innovation in Network Function Control
We use a combination of replayed university-to-cloud [24] and datacenter [19] network traffic traces, along with synthetic workloads.

[19] T. Benson, A. Akella, and D. Maltz. Network Traffic Characteristics of Data Centers in the Wild. In IMC, 2010.  
https://irtf.org/raim-2015-papers/raim-2015-paper41.pdf  
[24] K. He, L. Wang, A. Fisher, A. Gember, A. Akella, and T. Ristenpart. Next stop, the cloud: Understanding modern web service deployment in EC2 and Azure. In IMC, 2013.  
http://pages.cs.wisc.edu/~akella/papers/cloudmeasure-imc13.pdf  


### NFP: Enabling Network Function Parallelism in NFV
For test traffic, we use a DPDK based packet generator that runs on a separate server and is directly connected to the test server.

We evaluate NFP based on real world service chains in data centers [32, 36] including service chains for north-south and eastwest traffic. For NFP policies, we assume the operator assigns a sequential chain description based on Order rules for neighboring NFs in the chain. We generate test packets according to the packet size distribution derived from [4].

[4] Theophilus Benson, Aditya Akella, and David A Maltz. 2010. Network traffic characteristics of data centers in the wild. In Proceedings of the 10th ACM SIGCOMM conference on Internet measurement. ACM, 267–280.  
https://irtf.org/raim-2015-papers/raim-2015-paper41.pdf  
[32] Dilip A Joseph, Arsalan Tavakoli, and Ion Stoica. 2008. A policy-aware switching layer for data centers. In ACM SIGCOMM Computer Communication Review, Vol. 38. ACM, 51–62.  
http://ccr.sigcomm.org/online/files/p51-josephA.pdf  
[36] S Kumar, M Tufail, S Majee, C Captari, and S Homma. 2015. Service Function Chaining Use Cases in Data Centers. IETF SFC WG (2015).  


### NFVnice: Dynamic Backpressure and Scheduling for NFV Service Chains

We make use of DPDK based high speed traffic generators, **Moongen** [12] and **Pktgen** [38] as well as **Iperf3** [11], to generate line rate traffic consisting of UDP and TCP packets with varying numbers of flows. Moongen and Pktgen are configured to generate 64 byte packets at line rate (10Gbps), and vary the number of flows as needed for each experiment.
Workload Heterogeneity. We next use 3 homogeneous NF's with the same compute cost, but vary the nature of the incoming packet flows so that the three NFs are traversed in a different order for each flow. We increase the number of flows (each with
equal rate) arriving from 1 to 6, as we go from Type 1 to Type 6, with each flow going through all 3 NFs in a random order. Thus, the bottleneck for each flow is different. Figure 12, shows that the native schedulers (first four bars) perform poorly, with degraded throughput as soon as we go to two or more flows, because of the different bottleneck NFs.
In this experiment, we generate TCP and UDP flows with Iperf3. One TCP flow goes through only NF1 (Low cost) and NF2 (Medium cost) on a shared core. 10 UDP flows share NF1 and NF2 with the TCP flow, but also go through an additional NF3 (High cost, on a separate core) which is the bottleneck for the UDP flows - limiting their total rate to 280 Mbps.
We first start the 1 TCP flow. After 15 seconds, 10 UDP flows start, but stop at 40 seconds. As soon as the UDP flows interfere with the TCP flow, there is substantial packet loss without NFVnice, because NF1 and NF2 see contention from a large amount of UDP packets arriving into the system, getting processed and being thrown away at the queue for NF3. The throughput for the TCP flow craters from nearly 4 Gbps to just around 10-30 Mbps (note log scale), while the total UDP rate essentially keeps at the bottleneck NF3’s capacity of 280 Mbps.

