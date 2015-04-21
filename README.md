# docknfv
docker based network function virtualization in OpenContrail

In OpenContrail Virtual Network Functions (VNFs) can be setup as a Service Instance (SI). A typical SI uses two virtual networks (VNs) for interconnecting networks from the left and the right side. The interconnection is achieved by adding a SI to a network policy which is attached to a left and a right VN and effectively leaking routes between the left and the right VN. Typical VNF SIs are firewalls, load-balancers, VPN gateways, IDS, IPS etc, basically everything which forwards traffic from one to the other side. Todays VNFs come in form of fully fledged virtual machines (VMs) running on top of a hypervisor (HV) providing their own operating system (OS) stack which comes at the cost of resources. 
A less resource intesive alternative to VMs are Linux Containers. Containers leverage the kernel and other resorces of the host OS. A Container uses Linux Namespaces to isolate the networking stack on a per Container basis which makes it a perfect candidate for a VNF as long as the kernel modules required by the VNF are available in the host OS kernel. Docker is a mechanism to distribute and manage Containers. For OpenContrail a Docker plugin exists allowing to create, start, stop and delete Containers.
This article presents two Docker based VNFs:
- Dockwall - a simple ufw/iptables based firewall
- Dockvpn - an OpenVPN based SSL VPN solution

