# Networking Day 1: OSI + IP/CIDR

## OSI model
- The 7 layers, bottom to top, with the job of each:
1.physical layer --> transmission of 0's and 1's in the form of radio,electrical signals over cable,wire,wifi
2.data link layer --> Adding MAC addresses of next device on same network
3. network layer --> Adding IP addresses of source and destination to transmit over the networks
4.Transport layer --> Uses TCP or UDP protocol to send and receive the data, adds ports as well
5. session layer --> responsible for initiating,maintaining and closing session between devices
6. presentation layer --> responsible for encryption,compression and format of the content
7. application layer -->   communicates to applications using respective protocols FTP,SMTP,HTTP/DNS/SSH

At data link layer L2 - frame
At network layer L3 -->  packet
At Transport layer L4 --> segment

At each router MAC keeps changing.based on that MAC address it find the next device to go on the same network. IP address does not change at any hop.
- Troubleshooting rule (which direction do I work, and why?):
While trouble shooting , we need to go from bottom to top,because each layer depends on the ones below it, so a lower-layer failure breaks everything above.

## IP addressing

- An IPv4 address is 32 bits, written as 4 octets, each 0 to 255
- What /n means: /n means number of bits fixed as network. For example, 10.0.0.0/24 means first 24 bits are fixed as network and remaining 32 - 24 bits are host bits.
 out of the 2^(host bits) addresses, 2 are reserved (network and broadcast).
- Hosts formula (classic):   AWS reserves 5 instead of 2. usable hosts = 2^(32 − n) − 2.so a /19 gives 8,187, not 8,190.

## Subnetting steps (my method)
for example, If I want 6 subnets for 10.0.0.0/16,
1. Bits to borrow:  3 
How I came to conclusion 3, check which 2 ^ which number will result in more than subnet count first.
 2 ^ 1 is 2 which is less than 6, 
 2 ^ 2 is 4 which is less than 6,
 2 ^ 3 is 8 which is first greater value than 6. so will borrow 3 bits.
2. New prefix: prefix = borrowed bits + /n , which is 3 + 16 = 19 here.
3. Host bits and addresses per subnet: 
Host bits = Total number of bits - new_prefix ==> 32 - 19 = 13.
addresses per subnet = 2 ^ host bits ==> 2 ^ 13 = 8192 .
4. Step size and which octet: step size = 2 ^ number of host bits left in the octet. The borrowed bits land in the third octet. It has 8 bits, 3 are borrowed, so 8 − 3 = 5 host bits are left in that octet. Step size = 2^5 = 32
5. Sanity check: sanity check is number of subnets * addresses per subnet = Total size


## Worked example: 10.0.0.0/16 into 6 subnets
- Why /19 and not /18 or /20:
if we borrow only 2 bits, then 2 ^ 2 is 4 which is less than 6,4 bits gives 16 subnets (/20), which works but is more than I need, and each subnet is only 4,096 addresses instead of 8,192
/20 gives each subnet only 4,096 addresses instead of 8,192.

- The 8 network addresses:
I will use the first 6 and keep the last 2 (10.0.192.0/19 and 10.0.224.0/19) spare.”
First subnet
network address  = 10.0.0.0/19
first usable is 10.0.0.1/19,
last usable is 10.0.31.254/19
broadcast is 10.0.31.255/19
second subnet:
network address  = 10.0.32.0/19
first usable is 10.0.32.1/19,
last usable is 10.0.63.254/19
broadcast is 10.0.63.255/19
Third subnet:
network address  = 10.0.64.0/19
first usable is 10.0.64.1/19,
last usable is 10.0.95.254/19
broadcast is 10.0.95.255/19
Fourth subnet:
network address  = 10.0.96.0/19
first usable is 10.0.96.1/19,
last usable is 10.0.127.254/19
broadcast is 10.0.127.255/19
Fifth subnet:
network address  = 10.0.128.0/19
first usable is 10.0.128.1/19,
last usable is 10.0.159.254/19
broadcast is 10.0.159.255/19
Sixth subnet:
network address  = 10.0.160.0/19
first usable is 10.0.160.1/19,
last usable is 10.0.191.254/19
broadcast is 10.0.191.255/19
Seventh subnet:
network address  = 10.0.192.0/19
first usable is 10.0.192.1/19,
last usable is 10.0.223.254/19
broadcast is 10.0.223.255/19
Eighth subnet:
network address  = 10.0.224.0/19
first usable is 10.0.224.1/19,
last usable is 10.0.255.254/19
broadcast is 10.0.255.255/19


## Python check (paste my code):

import ipaddress
net=ipaddress.ip_network('10.0.0.0/16')
for subnet in net.subnets(new_prefix=19):
        hosts=list(subnet.hosts())
        print(f"{subnet} network address: {subnet.network_address} first usable address: {hosts[0]},last usable address:{hosts[-1]},broadcast address: {subnet.broadcast_address}")

## My mistakes today (for next time)
- while counting network address, I tend to forget step size
- while writing python code, I did net.hosts instead of subnet.hosts(). 
-new_prefix in the code takes whole prefix instead of borrowed bits count.
- network_address and broadcast_address have underscore in their names
