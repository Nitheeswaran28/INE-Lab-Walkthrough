Identifying Network Configuration
Step 1: Analyze the network interfaces and IP address configuration of the system.

ip a 
Ubuntu :

Content Image

Kali attacker :

Content Image

This command shows IP addresses, interface status, MAC addresses and more.

As we can see in the output, we have two network interfaces: - lo → Loopback interface - ens5 → Main network interface

In kali attacker machine, we can observe there are two active network interfaces (eth0 and eth1). This means that the kali machine is connected to two different networks simultaneously.

Note : A machine must have at least one network interface to communicate on a network. Depending on its role, it may have additional interfaces to connect to different network segments such as private networks, management networks, or the internet.

Loopback interface - The loopback interface is a virtual network interface that allows a computer to communicate with itself.

The entire 127.0.0.0/8 IPv4 range is reserved for loopback, which means 127.0.0.1 – 127.255.255.254 are all loopback addresses.

Applications use loopback to talk to services running on the same machine.

Main network interface - The main network interface is the physical or virtual network adapter that connects your machine to: 1. Local Area Network (LAN)

Internet

Other devices

Now if we look at the output, link/ether 06:6d:e6:87:9f:b7 is the MAC address of this system. A MAC address is a unique hardware identifier assigned to a network interface card (NIC) for communication within a local network.

Below the MAC address, we can see the IPV4 address of the machine 10.0.17.231. IPv4 address is a 32-bit logical address used to identify devices on a network.

Below the IPV4 address, we can see the IPV6 address fe80::46d:e6ff:fe87:9fb7/64. IPv6 is the next-generation internet addressing system, since we ran out of IPV4 addresses.

Note : These addresses may be different for your machines.

Step 2 : Display Host IP Address

hostname -I
Ubuntu :

Content Image

Kali attacker :

Content Image

The hostname -I command displays the IP address assigned to the system (excluding the loopback address).

This command is useful when: - Quickly checking the IP address - Using scripts

Now in the Kali machine we have got two network interface, so each network interface gets its own IP address.

Step 3 : Display Routing Table

ip route 
This command is to view how the system forwards network traffic.
Ubuntu :

Content Image

In the first line we can see the default route 10.0.16.1. This is the route that system takes when no other route matches. In simple terms, it means that if our system wants to reach internet or connect to other network, it takes the default route.

In the second line, we can see the local network route. So if our system wants to talk to any IP inside that subnet, it will send traffic directly through ens5, NOT through the default route.

In the third line, we have the gateway direct route. This tell the system that the default route 10.0.16.1 is directly reachaable scope link on the local network through ens5.

Similary, we can the network traffic routing on the kali machine.

Kali attacker :

Content Image

Step 4 : View DNS Configuration

cat /etc/resolv.conf
This is for identifying DNS server the system uses for domain name resolution.
Ubuntu :

Content Image

This nameserver 127.0.0.53 is the most important line which tells us our DNS resolver. Now if we look at what we got, that is a loopback address which means, the system is not directly querying an external DNS server. It is querying a local DNS stub resolver which then forwards requests to real DNS server.

Now let's try and find out the real DNS server.

resolvectl status
Content Image
Now we see the real DNS server which is at 10.0.0.2

Let's checkout the DNS server in the Kali machine.

Kali attacker :

Content Image

The /etc/resolv.conf file shows that Kali is using 127.0.0.11 as its DNS server. This indicates the system is running inside a Docker container, where Docker provides an internal DNS resolver. The container forwards DNS queries to the host’s configured DNS server. In containerized environments, DNS resolution may differ from traditional systems

Ports and Services
Step 1 : Identify Listening Services.

ss -tulnp
Content Image
The ss command is used to display network sockets on the system. It shows which services are listening, which ports are open, and which processes own them.
tulnp is the command options here. t -> TCP ports ; u -> UDP ports ; l -> Listenning ports only ; n -> show numerical addresses ; p -> show processes using the ports.
We can see in the output, we have got service sshd running at port 22.
Step 2 : Perform Network scan using Nmap.

From the kali machine, perform

nmap -sS 10.0.17.231
- Nmap (Network Mapper) is a network scanning tool used to discover live hosts, open ports, and running services on a target system. - It is commonly used for security auditing and reconnaissance to identify exposed services and potential attack surfaces. - -sS tells Nmap to perform a Half-Open Scan, here the connection is never fully established.
Content Image

Now in the output, we can see that the host is up meaning target machine is reachable.

Also out of 1000 common ports scanned, 996 responded with RST (reset) meaning these ports are closed.

Closed means the firewall is not blocking — the service simply isn’t running.

Below that we can see the open ports identified, along with the services that are running on those ports.

Step 3 : Mapping ports with services

Port 22/TCP is mapped to the SSH service, which allows secure remote login and administration of the system.

Port 80/TCP is mapped to the HTTP service, indicating that the Apache web server is running and serving web content on the default web port.

Port 3389/TCP corresponds to RDP (ms-wbt-server), which is used for remote desktop access.

Port 5910/TCP is associated with a remote access service (likely VNC), representing an additional remote management interface and expanding the system’s attack surface.

Network Traffic Generation
Step 1 : Perform basic connectivity testing.

From your kali machine

ping -c 4 10.0.27.36
Content Image
ping is a network utility used to test connectivity between two systems.
The -c 4 option tells ping to send 4 packets only.
Ping works using the ICMP (Internet Control Message Protocol) which is a Network layer protocol used for error reporting and diagnostics.
When we run ping command, our system send the ICMP echo request. If the target is reachable, it replies with ICMP Echo Reply.
Step 2 : Perform SSH connection test.

From the kali machine

ssh sshuser@10.0.27.36
Password : [redacted]
Content Image

SSH (Secure Shell) is a secure remote access protocol used to log into and manage systems over a network. It provides secure login, encrypted command execution, secure file transfer and remote server management.

Let's now go over the ubuntu machine and checkout the authentication logs generated

sudo tail -f /var/log/auth.log
Content Image
sshd[9171]: Accepted password for sshuser from 10.10.38.19 port 41194 ssh2
This line in the log tell us : - Accepted password : Login successful - sshuser : Username used - from 10.10.43.13 : Source IP - port 45564 : Source port (ephemeral) - ssh2 : Protocol version
Step 3 : Test HTTP Service & Analyze Web Server Logs From the kali machine

curl http://10.0.17.231/
curl http://10.0.17.231/admin
Content Image

This sends an HTTP GET request to the web server.

Let's now observe the apache web logs on ubuntu

sudo tail -f /var/log/apache2/access.log
Content Image

Here in the logs, we can see the source IPs from where the requests have been made. We can also see that curl was used make those requests and the endpoints on which they were made.

Step 4 : Simulate a bind shell using Netcat

On Ubuntu:

nc -lnvp 4444
This command starts Netcat in listening mode on port 4444. The -l option enables listener mode,-n for no DNS, -v enables verbose output, and -p 4444 specifies the port number. The system is now waiting for an incoming TCP connection on port 4444.
Content Image

Content Image

We can see that a process has started that is in Listen state and is waiting for a network connection.

From Kali:

nc 192.168.56.10 4444
This command initiates a TCP connection to the target system on port 4444.
Content Image

Content Image

We can see that the connection is established between both the systems.

Content Image

Now the connection is made, a new socket enters ESTABLISHED state while the original socket remains in LISTEN mode. This demonstrates how TCP handles active connections while continuing to wait for new ones.

Packet Capture and Inspection
Step 1 : Open Wireshark

Content Image

Step 2 : Go to capture tab and click on start.

Content Image

This will start capturing the packets.

Step 3 : Go to kali machine and run nmap command on the ubuntu machine to generate traffic packets.

Content Image

Step 4 : Stop the capture and apply a display filter and check the packets captured

Content Image

The filtered traffic shows packets generated by an Nmap SYN scan, where multiple TCP SYN packets are sent to different destination ports on the target host to identify open services. The presence of SYN packets followed by SYN-ACK (open ports) or RST responses (closed ports) indicates active reconnaissance activity against the system.

Step 5 : Let's now simulate a SSH brute force attack.

start the capture
On kali machine

ssh sshuser@10.0.17.231
[Password-guessing instruction omitted]
Content Image

Step 6 : Stop the capture and apply a new display filter.

tcp.port == 22
Content Image
The filter tcp.port == 22 isolates SSH traffic, showing the TCP handshake followed by SSH key exchange and encrypted communication between the client and server. Here we can see the IP of the source and if we see scroll down, we can see multiple loginn attempt were made

Step 7: Let's now perform directory scan using dirb and scan the traffic.

On Wireshark

Start the capture 
On Kali machine

[Automated directory enumeration command omitted]
Content Image
The dirb command performs a web directory brute-force scan against the target web server, attempting to discover hidden files and directories. It sends multiple HTTP requests using a wordlist, which we can observe in Wireshark

Step 8: Stop the capture and apply a new display filter.

http
Content Image
The captured traffic shows multiple HTTP GET requests generated by the dirb directory scan, attempting to access various file and directory paths on the target web server. The repeated 404 (Not Found) and 403 (Forbidden) responses indicate automated enumeration activity rather than normal user browsing behavior.

Step 9: Right click on any packet and follow http stream.

Content Image

Content Image

The “Follow HTTP Stream” option reconstructs the full request and response exchanged between the client and server. This allows us to analyze the exact HTTP GET requests, headers, and server responses associated with the activity.

Firewall Logging & Network Visibility
Step 1 : Enable Firewall Logging

sudo ufw enable
Content Image

This sudo ufw enable command will turn on the uncomplicated firewall on the machine, which will enforce firewall rules.

Step 2 : Check the firewall status

sudo ufw status verbose
Content Image
It displays the current status and configuration of the UFW firewall, including: - Whether it is active or inactive - Default policies (allow/deny) - All configured rules - Logging level

Step 3 : Let's now simulate Blocked Inbound Traffic.

From the kali machine :

nc -v 10.0.17.231 23
Content Image
Here we uses Netcat (nc) to attempt a TCP connection to our Ubuntu machine at port 23 (Telnet).

Step 4 : Inspect the logs generated.

cat /var/log/ufw.log
Content Image

Let’s break down the important log fields from your UFW log clearly.

[UFW BLOCK] IN=ens5 OUT= MAC=...
SRC=10.10.38.19 DST=10.0.17.231 LEN=60
PROTO=TCP SPT=53958 DPT=23
SRC=10.10.38.19 : This is the IP address that initiated the connection (potential attacker).
DST=10.0.17.231 : This is the target system.
PROTO=TCP : This is the communication protocol used.
DPT=23 : This is the port attacker is trying to access. In this case, it is port 23. This will help in identifying what service was targeted.
SPT=53958 : This is the ephemeral port used by the attacker.
Conclusion
This lab demonstrated how network configuration, service exposure, and system activity can be analyzed from both a system-level and network-level perspective. By capturing live traffic, reviewing authentication logs, analyzing web server logs, and inspecting firewall events, learners gained practical experience in identifying reconnaissance, brute-force attempts, enumeration behavior, and blocked connection attempts. Building strong foundational knowledge in traffic analysis and log correlation is essential for any SOC analyst operating in modern enterprise environments.