# Networking Day 10: TCP/UDP, ports, NAT, firewalls, load balancing

## TCP vs UDP
- Difference, with one use for each:
TCP is like registered courier , It will be slow, acknowledgment on both sides and guarantee that each piece of information is delivered. duplicates won't be sent. It should be used for important data and each piece of information is important
UDP is like post, It will be fast, no acknowledgment just send and hope it will be reached.It should be used where late arrival of packet is not an issue.
- 3-way handshake:
TCP uses 3-way handshake 
client --> server SYN          --> hello are you there? 
server --> client SYN-ACK      --> yes I am here
client --> server ACK          --> ok great

## Ports
- What a port is, and the 4 well-known ones (22, 80, 443, 53):
port is a number that identifies which program gets the traffic , It is like a door number in an apartment(IP address). Using IP address information will reach the machine but inside machine port tells which process handles that request.
port is from 0 - 65535
0 - 1023 is predefined ports
1024 - 49151 for application ports like mysql
49152 - 65535 is for machine to use as it's own source port when acting as client
Linux actually uses 32768 to 60999

- Server port vs client port (how to tell):
server will listen on port for request
client will send request to that ports on the server using any of the 49151 - 65543 ports as random source port

## Lab: commands I ran
- ss -tulpn (how to read 0.0.0.0 vs 127.0.0.1):
ss is socket statistic it shows the incoming and outgoing connections of ports using which protocol. 0.0.0.0 means any machine from outside can access that port if the firewall allows it. 127.0.0.1 means that only programs on the same machine can access.
ss lists sockets: listeners (-l) and active connections.
- nc -l / nc localhost (what I proved):
nc -l means asking a server machine to listen on 9000. nc localhost means asking client to connect on this port. I tested between two terminals on the same machine through localhost, not two machines.
- ufw allow / delete (what changed, and why both firewalls matter):
basically aws instances have two firewalls. by default both firewall block all incoming requests. security group is one and ufw is another one. both needs to be passed in order to receive information or requests

- traceroute (how to read a line, what * * * means):

each packet has a TTL that drops by 1 at every router. When it reaches 0, that router drops it and sends an error back, which is how traceroute finds each hop.
 * * * means (a hop that didn't reply, often normal) and that the time on each line is a round trip.

## Firewalls
- Security group vs NACL (3 differences):
Security group: allow rules only, stateful, attached to the instance.
NACL: allow and deny rules, stateless, attached to the subnet.
- Timeout vs refused:
Timeout means firewall is not allowing , refused means service/port is not listening for request

## NAT
- What it does, with one example:
With a NAT Gateway, outbound works and unsolicited inbound is blocked
many private devices share one public IP.
Network address Translator will translate private IP address to public IP address 

## Load balancing

- Health checks, and L4 vs L7:

 Health checks --> the load balancer regularly checks each server and stops sending traffic to one that fails.

L4 is Networking load balancer, It chooses tcp/udp, It is fast
L4 decides by IP and port only. AWS Name - NLB
 L7 load balancer reads the HTTP request (URL path, headers, host name), so it can send /api to one group of servers and /images to another
L7 is application load balancer, It processes https requests. It is slow
L7 AWS Name - ALB

## My mistakes today
The "My IP" button filled in the office laptop's address, not the personal laptop's.
Thinking ufw overrides the security group. They're two separate gates, and both must allow.
Security group vs NACL. I listed similarities and reversed the differences.
The ufw rule order. A deny after an allow for the same port does nothing,so I must delete the allow,.
Note:
ufw was off until I enabled it, and I allowed port 22 first so I wouldn't lock myself out of SSH.
