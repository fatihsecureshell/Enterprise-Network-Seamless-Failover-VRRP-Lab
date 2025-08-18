# About The Topology and introduction 
In this topology, I used vrrp on both vyos-1 and vyos-2 in the ether2 and ether2.10 interfaces for load balancing between subnets and **to provide network redundancy** for seamless failover if one router becomes unavailable.Here's the topology: 
![Topology](https://github.com/fatihsecureshell/gns3labarcive/blob/main/topology.png)
# Initial Verifications
The following commands were used to verify that all links and services are operational before the failover test:
## VRRP Status

To verify the VRRP configuration and the initial state of the network, I used the **show vrrp** command on **VyOS-1**. As expected, the router is in a **BACKUP** state for the 10.0.1.0/24 network and a **MASTER** state for the 10.0.2.0/24 network. This is shown in the output below:  
10.0.1.254  eth2             10  BACKUP           10  7h49m54s  
10.0.2.254  eth2.10          20  MASTER           20  7h49m50s  
Packets coming from the 10.0.2.0/24 network are then routed to **VyOS-3**. I simulated the ISP's network with a loopback interface, but in an enterprise network, you can connect **VyOS-3** directly to the ISP's router.
As expected, the **show vrrp** output from **VyOS-2** confirms that its VRRP states are the opposite of **VyOS-1's**. This ensures that one router is always in the **BACKUP** state for each subnet's gateway.Here's the output:  
10.0.1.254  eth2             10  MASTER           20  7h50m39s  
10.0.2.254  eth2.10          20  BACKUP           10  6h32m21s  
I checked this because a common problem in redundant setups is a "split-brain" scenario, where both devices incorrectly believe they are the **MASTER**. It is critical to verify the **BACKUP** state to ensure proper redundancy and avoid network issues.
## Packet Flow Analysis
To verify the routing path, a traceroute was performed from a client in the 10.0.1.0/24 subnet to an external destination.

traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 46 byte packets  
 1  10.0.1.252 (10.0.1.252)  3.767 ms  3.967 ms  1.599 ms  
 2  203.0.113.6 (203.0.113.6)  3.572 ms  4.149 ms  2.912 ms  
 3  1.1.1.1 (1.1.1.1)  2.572 ms  3.753 ms  2.392 ms  

---

### **Output Breakdown**

- **Hop 1 (10.0.1.252):** The packets first pass through **VyOS-2**, which is the master router for the 10.0.1.0/24 subnet.  
- **Hop 2 (203.0.113.6):** The traffic then proceeds to the next hop, which is the IP address of **VyOS-3**.  
- **Hop 3 (1.1.1.1):** Normally,your packets should be routed after **VyOS-3** receives it.But since it's an isolated network lab,I will continue like they did.  
### Asymmetric Path Verification

To fully demonstrate the asymmetric routing, a traceroute was also performed from a client in the 10.0.2.0/24 subnet. As expected, the traffic is routed through the master VRRP router for this subnet, which is **VyOS-1**.  

traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 46 byte packets  
 1  10.0.2.252 (10.0.2.252)  3.338 ms  3.260 ms  2.083 ms  
 2  203.0.113.2 (203.0.113.2)  1.945 ms  3.667 ms  2.279 ms  
 3  1.1.1.1 (1.1.1.1)  2.059 ms  4.469 ms  2.556 ms  
As I said,the 10.0.2/24 network first hopes to **Vyos-1**'s eth2.10 interface and then hops to **Vyos-3**.It doesn't hop to the route of the 10.0.1.0/24 network,thanks to the subinterface+vrrp configuration.  
### Which path is client pc's using to ping internet??
