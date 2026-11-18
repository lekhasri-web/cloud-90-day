# Networking Day 9: DNS + HTTP/S

## DNS
- What problem it solves:
Humans remember domain names, not IP addresses. DNS translates a domain name into an IP address, which the browser then uses to connect over TCP.
- The lookup chain (who asks whom, in order):
First my device checks its own cache. If the answer isn't there, it asks the resolver, which checks its own cache too. If the resolver doesn't have it, the resolver asks a root server, which points it to the .com servers, which point it to the authoritative server for the domain. The authoritative server returns the IP address, and the resolver passes it back to my device.
- Where the answer gets cached, and what TTL means:
The domain owner sets the TTL on the authoritative server. TTL (time to live) is how many seconds a cache may remember the answer. The resolver and my device both cache it. When the TTL runs out, the resolver discards its copy and asks the authoritative server again.
- Record types (A, AAAA, CNAME, NS, MX, TXT): one line each
A --> name to an IPv4 address
AAAA --> name to an IPv6 address
CNAME --> an alias: another name that points to the original name
NS --> the authoritative servers for the domain
MX --> where mail sent to the domain is delivered; the number is the priority, lowest is tried first, so extra servers give failover
TXT --> free text, used for domain verification and email security
- Which OSI layer and which port:
DNS is a Layer 7 (application) protocol on port 53, normally over UDP.

## HTTP / HTTPS
- Request parts and the 4 methods:
A request has a method, a path and headers.
GET --> fetch something from the server
POST --> create or send new data to the server
PUT --> update something on the server
DELETE --> delete something on the server
- Status code families (2xx / 3xx / 4xx / 5xx) with one example each:
2xx --> success (200 OK)
3xx --> redirect to another URL (301 moved permanently); not an error
4xx --> client failures, check the request (404 not found)
5xx --> server failures, check the server (503 unavailable)
- What TLS gives you, and the 3 certificate checks:
TLS gives encryption (nobody in between can read the traffic) and identity (I'm talking to the real site, not an imposter).
The 3 checks: the name matches, the certificate is signed by a trusted CA, and it hasn't expired.

## Full page load, in order
1. DNS: the browser's name is checked in the cache, then the resolver walks root, .com and the authoritative server to get the IP address.
2. TCP: connects to port 443 on the IP address received from DNS.
3. TLS: checks that the name is correct, the CA is trusted, and the certificate hasn't expired.
4. HTTP: the application layer sends a request (GET, POST, PUT or DELETE) and receives the response.
- One thing that can go wrong at each stage:
DNS --> the authoritative server doesn't respond, or the name is wrong
TCP --> the server is down, nothing is listening on the port, or a security group/firewall blocks port 443
TLS --> the name is wrong, the CA isn't trusted, or the certificate has expired
HTTP --> 4xx client errors or 5xx server errors (a 3xx is only a redirect, not a failure)

## Lab: commands I ran
- dig takes a domain name, not a URL. curl takes the full URL.

dig amazon.com A +noall +answer
output: amazon.com.             300     IN      A       98.87.170.74
--> shows name, TTL, class, record type and value.
TTL 300 means a cache can remember the answer for 300 seconds.
A means it's an IPv4 address; AAAA would be IPv6.
Running dig again showed the TTL counting down, which proved the caching.

- `curl -I` and `curl -v` (how I found each stage in the output):
curl -I https://example.com
Shows only the response headers and status code instead of the full page.
curl -v https://example.com -o /dev/null
-v (verbose) prints every stage as it happens. -o /dev/null throws away the page body so only the stages show.
Lines starting with * are curl telling me what it is doing, > is the request I sent, and < is the response I got back.

Where I found each stage:


DNS --> Host example.com:443 was resolved.
* IPv6: 2606:4700:10::ac42:93f3, 2606:4700:10::6814:179a
* IPv4: 172.66.147.243, 104.20.23.154                                       --->proves DNS got the IP address
*   Trying 172.66.147.243:443...
TCP --> Connected to example.com (172.66.147.243) port 443                  ---> Proves TCP connects to port 443.
TLS --> 
Server certificate:
*  subject: CN=example.com                                                 --->proves TLS got the name and verifies it
*  start date: Sep 26 22:49:11 2026 GMT                                            
*  expire date: Dec 25 22:56:35 2026 GMT                                   --->proves TLS receives and checks expiry date
*  subjectAltName: host "example.com" matched cert's "example.com"								
*  issuer: C=US; O=SSL Corporation; CN=Cloudflare TLS Issuing ECC CA 3     --->Proves it checks the Issue CA authority who issued the certificate
*  SSL certificate verify ok.											   --->proves the name, CA and expiry checks all passed. 
HTTP request --> > GET / HTTP/2                                            --->ProvesHTTP request is sent
> Host: example.com
> User-Agent: curl/8.5.0
> Accept: */*
> 
                                                                    
HTTP response --> < HTTP/2 200                                             --->Proves HTTP response is received
< date: Wed, 18 Nov 2026 13:17:27 GMT
< content-type: text/html; charset=utf-8
< server: cloudflare
< last-modified: Mon, 16 Nov 2026 00:30:20 GMT
< allow: GET, HEAD
< accept-ranges: bytes
< age: 3764
< cf-cache-status: HIT
< cf-ray: a5c7416839bd0c64-SYD
< alt-svc: h3=":443"; ma=86400


## My mistakes today (for next time)

1. 443 is a port number, and HTTPS uses TCP to connect to it. The transport layer (L4) is where TCP and UDP live.
2. MX and NS are different. NS is authoritative servers and MX is mail sent to the domain is delivered
3. Ping uses ICMP means if ping works then L3 and below layers are working not TCP or ports working
4.dig takes domain name only not whole url
