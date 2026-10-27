# Day 2: Users, groups, permissions, sudo

**Date:** 27 Oct 2026 | **Week 1, Day 2** | **Hours:** ~2.75

## Summary (3 lines)

- **Built:** Shared team folder `/srv/project` (owner `root:devs`, mode `770`), users `alice`, `bob`, `eve`, group `devs`, and a limited sudo rule for bob in `/etc/sudoers.d/bob`.
- **Broke:** 5 permission scenarios (listed below).
- **Fixed:** Each one with `chmod`, `chgrp` or `chown`, and verified as a normal user.

## Concepts learned

- Linux checks **numbers** (UID/GID), not names. UID 0 is root.
- Accounts live in `/etc/passwd`, groups in `/etc/group`, password hashes in `/etc/shadow`.
- `ls -l` string: type + owner/group/others, each as `rwx`. Check order: owner, then group, then others.
- On a **file**: `r` read, `w` modify, `x` run. On a **directory**: `r` list names, `w` create/delete entries, `x` enter (`cd`).
- Octal: r=4, w=2, x=1. Common modes: `644` files, `755` scripts/public dirs, `600` private files, `700` private dirs.
- `chmod`: root or owner. `chown` (change owner): **root only**. `chgrp`: root, or owner to a group they belong to.
- `sudo` runs one command as root using **your own** password. Rules are additive, so the `sudo` group's "any command" rule overrides a narrow rule.
- `usermod -aG` appends a group. Without `-a`, it **replaces** all supplementary groups.
- Use `visudo` (syntax check) and put custom rules in `/etc/sudoers.d/`, never edit `/etc/sudoers` with a plain editor.

## Lab A: Shared team folder

```bash
groupadd devs
useradd -m alice; useradd -m bob; useradd -m eve
usermod -aG devs alice
usermod -aG devs bob
mkdir /srv/project
chown root:devs /srv/project
chmod 770 /srv/project
```

Verification:

```bash
sudo -u alice touch /srv/project/alice.txt   # works
sudo -u bob   touch /srv/project/bob.txt     # works
sudo -u eve   touch /srv/project/eve.txt     # Permission denied
```

### Limited sudo for bob

```bash
gpasswd -d bob sudo                     # remove unlimited sudo first
visudo -f /etc/sudoers.d/bob
# rule: bob ALL=(root) NOPASSWD: /usr/bin/apt update
sudo -u bob sudo -n /usr/bin/apt update  # works
sudo -u bob sudo -n /usr/bin/whoami      # refused: a password is required
```

## Lab B: 5 break/fix scenarios

| # | Broke | Error | Fix |
|---|---|---|---|
| 1 | Script created without execute permission | `Permission denied` on `./hello.sh` | `chmod u+x hello.sh` |
| 2 | Removed all permissions with `chmod 000` | `cat: secret.txt: Permission denied` (only visible as a normal user) | `chmod 600 secret.txt` |
| 3 | Directory without `x` (`chmod 600 ~/vault`) | `Permission denied` on `cd` and on reading files inside | `chmod 700 ~/vault` |
| 4 | File in shared folder owned by alice's personal group, not `devs` | bob: `Permission denied` | `chgrp devs /srv/project/plan.txt` |
| 5 | Root created a file inside alice's home (owned by root) | alice could read but not write (`Permission denied` on append) | `chown alice:alice /home/alice/notes.txt` |

## Mistakes and lessons

- I was logged in as **root**, so root bypassed permission checks and hid the errors. Test as a normal user with `su - alice` or `sudo -u alice`.
- Even root cannot run a file with no `x` bit for anyone.
- Being in a group is not enough: the **file's group** must be that group.
- Typos in paths (`/usr/bin`, `demo-backup`) matter. Use Tab completion and run `ls` before `rm -r`.

## Recall (Day 1 + Day 2)

`man <cmd>` and `q` to quit, `cp -r`, `/etc` vs `/home`, `rm -r` with care as root, octal `640`, `visudo`, who may run `chmod` and `chown`.

