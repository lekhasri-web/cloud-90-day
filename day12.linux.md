# Networking Day 12: Troubleshooting layer by layer

## The method
- Work **bottom to top**, because each layer depends on the ones beneath it.
- **Stop at the first step that fails.** That's where the problem is.
- A step that **passes** also tells me something: it rules that layer out.
- Change **one thing at a time** and check after each change.
- Read **how** it fails: instant failure means the service, a hang or timeout means a firewall, and "permission denied" means file or user rights.

| Step | Question | Tool | If it fails |
|---|---|---|---|
| 1 | Is my network working? | `ping 8.8.8.8` | Network or routing |
| 2 | Does the name resolve? | `dig <name>` | DNS |
| 3 | Can I reach the machine? | `ping <ip>` | L3 or routing |
| 4 | Is the port open from here? | `nc -vz <ip> <port>` | Firewall (security group or `ufw`) |
| 5 | Is anything listening? | `ss -tulpn` | The service |
| 6 | Does the app work? | `curl -v`, logs | Application or permissions |

## DNS troubleshooting
- `dig` status values: `NXDOMAIN` (name doesn't exist), `SERVFAIL` (resolver couldn't get a valid answer), a timeout (can't reach the resolver), `NOERROR` (worked).
- `dig <name> @8.8.8.8` skips my local resolver. If it works and a normal lookup fails or differs, the problem is local.
- My instance's resolver is `172.31.0.2` (the VPC base address plus 2). I found it with `resolvectl status`. Programs ask the local stub at `127.0.0.53`, which forwards to it.
- Private names such as `.internal` only resolve through the VPC resolver, so a public resolver finding nothing proves nothing.
- Lookup order is in `/etc/nsswitch.conf`. Mine says `hosts: files dns`, so `/etc/hosts` is checked **before** DNS and wins.
- `getent hosts <name>` shows what programs actually receive. `dig @8.8.8.8` shows what real DNS says. A mismatch means something local is overriding DNS.

## Lab: three break/fix scenarios
**1. DNS** (bad line in `/etc/hosts`)
- Backed up first: `sudo cp /etc/hosts /etc/hosts.bak`
- Break: `echo "127.0.0.1 example.com" | sudo tee -a /etc/hosts`
- Symptom: `curl -s` printed nothing. Without `-s`, `curl -v` showed "Connection refused" to `127.0.0.1:443`.
- Diagnosis: `ping 8.8.8.8` worked, `dig @8.8.8.8` returned real IPs, `getent` returned `127.0.0.1`, and `grep example /etc/hosts` found the line.
- Fix: `sudo cp /etc/hosts.bak /etc/hosts`, then confirmed with `grep`, `getent` and `curl`.

**2. Port** (service stopped)
- Setup: `python3 -m http.server 8080 --bind 127.0.0.1`
- Break: stopped the server with Ctrl+C.
- Symptom: `curl http://localhost:8080` failed instantly (`after 0 ms`), and `ss | grep 8080` printed nothing.
- Diagnosis: instant failure means the service, not a firewall.
- Fix: restarted the server, then checked with `ss` and `curl`.

**3. Permission** (execute bit removed)
- Break: `chmod -x hello.sh`
- Symptom: `-bash: ./hello.sh: Permission denied`
- Diagnosis: `bash hello.sh` still worked, so the script is fine and readable, and only execute is missing.
- Fix: `chmod u+x hello.sh` (owner only, the smallest change that works).

## Useful commands
- `getent hosts <name>`: what programs get for a name
- `grep ^hosts /etc/nsswitch.conf`: lookup order
- `resolvectl status`: which resolver the machine uses
- `sudo resolvectl flush-caches`: clear a stale local cache
- `ls -l <file>`: permissions as three sets (owner, group, others)
- `curl -v` instead of `curl -s` while troubleshooting, because `-s` hides errors

## Incident Log (3 entries)
TODO: link to my three Incident Log rows, or paste them in my own words.

## My mistakes today
- A trailing slash in `cp` (`/etc/hosts/`) made Linux treat a file as a directory.
- Typos in commands: `head =1` instead of `head -1`, and `curl -t` instead of `curl -i`.
- I read `bash hello.sh` working as "I can't access the file", when it proves the script and read permission are fine.
- I concluded "fake line" because 8.8.8.8 found nothing. For a private name, that proves nothing.
