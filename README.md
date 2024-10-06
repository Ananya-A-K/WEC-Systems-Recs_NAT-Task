# Simple-NAT-implementation
## Tasks completed:
1. Create a network topology consisting of:
      - Private LAN (192.168.10.0/24): At least one client node.
      -  Router: Acts as the NAT gateway with an external public IP.
      - The topology should look like this: |Client| <--------> |Router (NAT)| <--------> |Internet (Simulated Public Network)|
2. Implement NAT using Linux network namespaces and iptables to allow the private network (LAN) to access the simulated internet through the router's public IP.
3. [Bonus] Configure Port Forwarding:
      - Host a simple web server (using Python, Apache, etc.) on the LAN client.
      - Forward requests from the "internet" to the internal web server using the router's public IP.
4. [Bonus] Restrict outbound traffic from the LAN to allow only HTTP and HTTPS connections.  <br>
## TASK1:  
Topology: | Client (192.168.10.2/24) | <--------> | Router (NAT) (192.168.10.1/24) | <--------> | Internet (Simulated Public Network)|
### Create Network Namespaces:
Create a namespace named client and then another named router
```bash
ip netns add client 
ip netns add router
```
### Create Virtual Ethernet Pairs:                                                                                              
Create a link or a virtual cable and name the ends as veth-client and veth-router
```bash
ip link add veth-client type veth peer name veth-router
```
### Assign Interfaces to Namespaces:                                                                                  
Assign each end of the link or virtual cable to the specific namespace
```bash
ip link set veth-client netns client                                                                             
ip link set veth-router netns router
```
### Assign IP Addresses:                                                                                                         
For the client:                                                                                                                     
Assign ip to client 
```bash
ip netns exec client ip addr add 192.168.10.2/24 dev veth-client
```                          
Get the link up and running
```bash
ip netns exec client ip link set veth-client up
```                                                            
For the router (NAT gateway):                                                                                           
Assign ip to router
```bash
ip netns exec router ip addr add 192.168.10.1/24 dev veth-router
```                                
Get the link up and running
```bash
ip netns exec router ip link set veth-router up
```
<!-- For loopback: ip netns exec client ip link set lo up -->
### Create a Simulated Public Network:
<!--
Create a virtual cable or link with ends veth-public and veth-router-public(link with router), asign ip and get it up and running
```bash
ip link add veth-public type veth peer name veth-router-public
ip link set veth-router-public netns router
ip netns exec router ip addr add 203.0.113.1/24 dev veth-router-public
ip netns exec router ip link set veth-router-public up
ip link set veth-public up
```
(multicast address: 203.0.113.1/24 is used to simulate public internet)
-->
<!--Loopback on the router namspace is used simulate the internet/public network, using public ip 192.0.2.1/24 for the router.
```bash
sudo ip netns exec router ip addr add 192.0.2.1/24 dev lo
sudo ip netns exec router ip link set lo up
```
-->
A namespace named "internet" is created with a public ip(198.51.100.0/24) and link is made between the router and internet. Loopback is used to simulate the traffic going through various nodes and finally reaches the router of the internet, finally to be sent to destination router. Public ip of the router is set as 192.0.2.1/24.
```bash
ip netns add internet
ip link add veth-router-public type veth peer name veth-internet
ip link set veth-router-public netns router
ip link set veth-internet netns internet
ip netns exec router ip addr add 192.0.2.1/24 dev veth-router-public
ip netns exec internet ip addr add 192.0.2.2/24 dev veth-internet
ip netns exec router ip link set veth-router-public up
ip netns exec router ip link set veth-internet up
sudo ip netns exec router ip addr add 192.168.10.1/24 dev lo
sudo ip netns exec router ip link set lo up
```
Note: 192.0.2.1/24 and 198.51.100.0/24 are used as public ip for the router as it belongs to the group of ips reserved for documentaion and testig purposes, to avoid confilcts or issues if actual/valid public ips. Alternatively, 198.18.0.0/15 block can be used for the same purpose.
Also, the private ip of the "internet" router is 192.168.10.1/24
## TASK2:
### Enable IP Forwarding: 
Enable IP forwarding on the router, allowing it to route packets between the client and the simulated public network.
```bash
ip netns exec router sysctl -w net.ipv4.ip_forward=1
```
Alternatively,
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```
The above two methods do not make the changes persistent.
To make the changes persistent, apply changes to the /etc/sysctl.conf file(nano editor used here).
```bash
nano /etc/sysctl.conf
```
After editing the file, to make the changes take effect right away.
```bash
sysctl -p
```
### Set up NAT on the router: 
Sets up a NAT rule in the router namespace that will perform IP Masquerade on packets leaving the router namespace, allowing the client to access the simulated public network through the router's public IP.
```bash
ip netns exec router iptables -t nat -A POSTROUTING -o veth-router-public -j MASQUERADE
```
<!--
NAT for incoming traffic to reach the right namespace
```bash
ip netns exec router iptables -t nat -A PREROUTING -i veth-router-public -j ACCEPT
```
Router forwards the traffic to the destination namespace
```bash
ip netns exec router iptables -A FORWARD -d 192.168.10.2 -j ACCEPT
```
-->
###Ping to test the working or connectivity.
```bash
ip netns exec client ping 192.0.2.1
```
## TASK3:
### Host a simple web server (using Python) on the LAN client: 
A simple web server is hosted using Python's built-in http.server module.
```bash
ip netns exec client python3 -m http.server 80
```
### Forward requests to the web server
```bash
ip netns exec router iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT –-to-destination 192.168.10.2:80
```
Forwarding all types of traffic
```bash
ip netns exec router iptables -A FORWARD -d 192.168.10.2 --dport 80 -j ACCEPT
```
## TASK4:
### Allow only HTTP(port 80) and HTTPS(port 443) traffic
```bash
ip netns exec router iptables -A FORWARD -p tcp --dport 80 -j ACCEPT
ip netns exec router iptables -A FORWARD -p tcp --dport 443 -j ACCEPT
ip netns exec router iptables -A FORWARD -j DROP
```
## To remove the namespaces and virtual cables/links after completion
```bash
ip netns del client
ip netns del router
ip netns del internet
ip link del veth-client
ip link del veth-router
ip link del veth-router-public
ip link del veth-internet
```
