# docknfv
docker based network function virtualization in OpenContrail

In OpenContrail Virtual Network Functions (VNFs) can be setup as Service Instances (SIs). A typical SI uses two virtual networks (VNs) for interconnecting networks from the left and the right side. The interconnection is achieved by adding a SI to a network policy which is attached to a left and a right VN and effectively leaking routes between the left and the right VN. This forces traffic between left and right to go through the SI. Typical VNF SIs are firewalls, load-balancers, VPN gateways, IDS, IPS etc, basically everything which forwards traffic from one to the other side. Todays VNFs come in form of fully fledged virtual machines (VMs) running on top of a hypervisor (HV) providing their own operating system (OS) stack which comes at the cost of resources. 
A less resource intesive alternative to VMs are Linux Containers. Containers leverage the kernel and other resorces of the host OS. A Container uses Linux Namespaces to isolate the networking stack on a per Container basis which makes it a perfect candidate for a VNF as long as the kernel modules required by the VNF are available in the host OS kernel. Docker is a mechanism to distribute and manage Containers. For OpenContrail a Docker plugin exists allowing to create, start, stop and delete Containers.

This article presents two Docker based VNFs:
- Dockwall - a simple ufw/iptables based firewall
- Dockvpn - an OpenVPN based SSL VPN solution

As initially said the VNFs connect to two VNs: left and right. As such the VNF has two network interfaces (NICs). Having two NICs (dual home) creates a challenge in terms of routing as only the directly attached VNs are known to the Container. But the Container must receive and send traffic to networks which are not directly attached. A typical routing device solves that problem by either using a default route on one NIC and static routes on the other or by using a dynamic routing protocol on both NICs. Dynamic routing protocols require a peer which revceives and advertises routing information. Whilst the Container could be configured with a dynamic routing protocol a per Container peer configuration would be required which cannot be provided by OpenContrail.
Setting static routes at the other hand means to create configuration inside Container which again cannot be done by OpenContrail. 
For the two examples two different approaches are used:

1. Dockwall:

Dockwall works with two routing tables inside the container, one for each side. Each routing table uses a default route to the VNs default gateway and a route policy. The route policy basically defines that each packet hitting one side will lookup the routing table of the other side and the visa versa. Because of default route set in that table the traffic will be delivered to the next hop. In between the two NICs the traffic can be processed by kernel functions such as netfilter. Userland tools such as iptables/ufw can easily create L3/L4 based firewall and nat rules. The Dockwall images can be instiated as a SI without any manual configuration and will route traffic between left and right side. Changing firewall/nat rules can be done by either ssh'ing into to the Container, providing an ufw/iptables web ui or by using Docker exec commands on the host.
Ideally at some point in the future the OpenStack Neutron FWaaS plugin is supported for configuring iptables.

<pre>
+----------------------------------------------------------------------+  
|                                                                      |  
|                               Compute Node                           |  
|                                                                      |  
|                                 +-----------------------------+      |  
|                                 |   Dockwall Container        |      |  
|                                 |                             |      |  
|                                 |                             |      |  
|        +------+  +------+   d + +---------+        +----------+ + d  |  
|        | VM22 |  | VM12 |   e | |         |        |          | | e  |  
|        | +----++ | +----++  f | | left RT | route  | right RT | | f  |  
|        | | VM21| | | VM11|  a | |         | policy |          | | a  |  
|        | |     | | |     |  u | | +----+ <----------> +-----+ | | u  |  
|        +-+     | +-+     |  l | | |eth1|  |iptables|  |eth0 | | | l  |  
|          +--+--+   ++----+  t | +---+---------------------+-+-+ | t  |  
|             |       |         |     |                     |     |    |  
| +---------------------------r---------------------------------+ | r  |  
| |           |       |       o |     |                     |   | | o  |  
| |        +--+--+ +--+--+    u |   +-+-------+    +--------+-+ | | u  |  
| |        |     | |     |    t |   |         |    |          | | | t  |  
| |        | VN2 | | VN1 |    e v   | left VN |    | right VN | | v e  |  
| |        |     | |     |          |         |    |          | |      |  
| |        +--+--+ +-----+----------++---+----+    +-----+----+ |      |  
| |           |                      |   |               |      |      |  
| |           +----------------------+   |   vRouter     |      |      |  
| |                                      |               |      |      |  
| +-------------------------------------------------------------+      |  
|                                        |               |             |  
+----------------------------------------------------------------------+  
                                         |               |                
                                  +-----------------------------+         
                                  |      |               |      |         
                                  | +----+----+     +----+----+ |         
                     Corporate NW | |         |     |         | | Internet
                       <------------+ Corp VRF|     |Inet VRF +---------->
                                  | |         |     |         | |         
                                  | +---------+     +---------+ |         
                                  |                             |         
                                  |        Router (DCGW)        |         
                                  |                             |         
                                  +-----------------------------+         
</pre>
In the example above the Dockwall Container forwards traffic between the Internet, Virtual Networks 1 and 2 and the Corporate Network.


2. Dockvpn: 

Dockvpn uses OpenVPN as the SSL VPN software. The approach from above with the two routing table and the route policy cannot be used as traffic has to go through a tun interface and cannot be blindley forwarded between both sides. So routes for both interfaces must be configured in the global routing table. The approach used here is to configure a default route on the right side (typically the side facing to the internet) and use Contrails vRouter to send rfc3442 classless routing information through dhcp for the left side NIC. The routes being sent are the ones which are reachable from the VPN clients connected to the Dockvpn instance. This approach has a very neat side effect: The OpenVPN server sends routes to the OpenVPN client. It either sends a set of specific routes or a default route, which effectively means that either all traffic or only traffic to selected destination is routed through the OpenVPN server. If the desire is to only send specific routes to the client the routes have to be configured in the OpenVPN server configuration. As we don't want to apply manual configuration changes to anything inside the Container it is possible to utilize the dhclient script to push the received routing information into the OpenVPN server configuration. Whenever a VPN client connects the new routing information will be present.

<pre>
+----------------------------------------------------------------------+  
|                                                                      |  
|                               Compute Node                           |  
|                                                                      |  
|                                 +-----------------------------+      |  
|                                 |      Dockvpn  Container     |      |  
|                                 |     +----------------+      |      |  
|                             d ^ |     |     OpenVPN    |      |      |  
|        +------+  +------+   h | |     +--------+-------+      | + d  |  
|        | VM22 |  | VM12 |   c | |              |              | | e  |  
|        | +----++ | +----++  p | |           +--+-+            | | f  |  
|        | | VM21| | | VM11|    | |    ┌------+tun0+-------┐    | | a  |  
|        | |     | | |     |  r | | +----+    +----+    +-----+ | | u  |  
|        +-+     | +-+     |  f | | |eth1|              |eth0 | | | l  |  
|          +--+--+   ++----+  c | +---+---------------------+-+-+ | t  |  
|             |       |       3 |     |                     |     |    |  
| +-------------------------------------------------------------+ | r  |  
| |           |       |       4 |     |                     |   | | o  |  
| |        +--+--+ +--+--+    2 +   +-+-------+    +--------+-+ | | u  |  
| |        |     | |     |          |         |    |          | | | t  |  
| |        | VN2 | | VN1 +----------+ left VN |    | right VN | | v e  |  
| |        |     | |     |          |         |    |          | |      |  
| |        +--+-++ +----++          +-+--+----+    +-----+----+ |      |  
| |           | |       |             |  |               |      |      |  
| |           +-----------------------+  |   vRouter     |      |      |  
| |             |       |                |               |      |      |  
| +-------------------------------------------------------------+      |  
|               |       |                |               |             |  
+----------------------------------------------------------------------+  
                |       |                |               |                
                |       |         +-----------------------------+         
                |       |         |      |               |      |         
                |       |         | +----+----+     +----+----+ |         
                |       +-----------+         |     |         | |         
                |                 | | Corp VRF|     |Inet VRF | |         
                +-------------------+         |     |         | |         
                                  | |         |     |         | |         
                                  | |         |     |         | |         
                       <------------+---------+     +---------+---------->
                      Corporate NW|                             | Internet
                                  |        Router (DCGW)       |         
                                  |                             |         
                                  |                             |         
                                  +-----------------------------+         
</pre>

The Dockvpn Container allows VPN clients to access Virtual Networks 1 & 2 and the Corporate Network.
