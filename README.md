# docknfv
docker based network function virtualization in OpenContrail

In OpenContrail Virtual Network Functions (VNFs) can be setup as a Service Instance (SI). A typical SI uses two virtual networks (VNs) for interconnecting networks from the left and the right side. The interconnection is achieved by adding a SI to a network policy which is attached to a left and a right VN and effectively leaking routes between the left and the right VN. Typical VNF SIs are firewalls, load-balancers, VPN gateways, IDS, IPS etc, basically everything which forwards traffic from one to the other side. Todays VNFs come in form of fully fledged virtual machines (VMs) running on top of a hypervisor (HV) providing their own operating system (OS) stack which comes at the cost of resources. 
A less resource intesive alternative to VMs are Linux Containers. Containers leverage the kernel and other resorces of the host OS. A Container uses Linux Namespaces to isolate the networking stack on a per Container basis which makes it a perfect candidate for a VNF as long as the kernel modules required by the VNF are available in the host OS kernel. Docker is a mechanism to distribute and manage Containers. For OpenContrail a Docker plugin exists allowing to create, start, stop and delete Containers.

This article presents two Docker based VNFs:
- Dockwall - a simple ufw/iptables based firewall
- Dockvpn - an OpenVPN based SSL VPN solution

As initially said the VNFs connect to two VNs: left and right. As such the VNF has two network interfaces (NICs). Having two NICs creates a challenge in terms of routing as only the directly attached VNs are known to the Container. But the Container must receive and send traffic to networks which are not directly attached. A typical routing device solves that problem by either using a default route on one NIC and static routes on the other or by using a dynamic routing protocol on both NICs. Dynamic routing protocols require a peer which revceives and advertises routing information. Whilst the Container could be configured with a dynamic routing protocol a per Container peer configuration would be required which cannot be provided by OpenContrail.
Setting static routes at the other hand means to create configuration inside Container which again cannot be done by OpenContrail. 
For the two examples two different approaches are used:
1. Dockwall:
Dockwall works with two routing instances inside the container, one for each side. Each routing instance uses a default route to the VNs default gateway and a route policy. The route policy basically defines that each packet hitting one side will be forwarded to the other side and the visa versa. The default routes take care that the traffic will be delivered to the next hop. In between the two NICs the traffic can be processed by kernel functions such as netfilter. Userland tools such as iptables/ufw can easily create L3/L4 based firewall and nat rules.
