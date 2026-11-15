# Day 6: Build Session (Log Monitor + Cron + SSH Keys + GitHub)

Saturday build session. Goal: combine Week 1 Linux skills into one small project.

## What I built

1. A bash script, `monitor.sh`, that counts `ERROR` lines in a log file
2. A cron job that runs it on a schedule
3. SSH key login to my EC2 instance from my own laptop
4. A GitHub repo with the script, pushed over SSH

## 1. The log monitor script

`monitor.sh` takes a log file path as the first argument and an optional threshold as the second argument.

What it does:
- If no file is given, or the file does not exist, it prints a usage message and exits with code `2`
- Counts lines containing `ERROR` with `grep -c`
- Appends one line to `monitor.out` with a timestamp, the count and the file name
- Exits `0` if the count is below the threshold, exits `1` if it is equal or above

The threshold defaults to 3 using `threshold=${2:-3}`. This means `./monitor.sh app.log` still works the same as before, and `./monitor.sh app.log 5` uses 5.

**How I tested it (exit codes with `echo $?`):**

| Command | Expected | Why |
|---|---|---|
| `./monitor.sh app.log` (2 errors in log) | 0 | 2 is below 3 |
| `./monitor.sh app.log` (3 errors in log) | 1 | `3 -lt 3` is false |
| `./monitor.sh app.log 5` | 0 | 3 is below 5 |
| `./monitor.sh` | 2 | no argument |
| `./monitor.sh /nope` | 2 | file does not exist |

Key commands: `grep -c`, `$(command)` to store output in a variable, `date '+%F %T'` for the timestamp, `>>` to append, `exit` codes.

## 2. Cron

My entry in `crontab -e`:

```
*/5 * * * 1-5 /home/ubuntu/linux-labs/monitor.sh /home/ubuntu/linux-labs/app.log
```

This means every 5 minutes, Monday to Friday.

The 5 fields are minute, hour, day of month, month, day of week.

Things I got wrong first and learned from:
- `5` in the minute field means "at minute 5 of every hour". `*/5` means every 5 minutes.
- Putting `0` in the hour field would only run during hour 0.
- `1-6` includes Saturday, so weekdays are `1-5`.

**Why cron needs absolute paths:** cron does not run in my home folder and has a very small environment. A relative path like `app.log` means "in the folder I am currently in", and that is not `~/linux-labs` under cron. Absolute paths mean the same thing everywhere.

I tested with `* * * * *` first and saw new lines in `monitor.out` landing on second `:01`, which is how I knew cron was running it and not me. Then I switched to `*/5 * * * 1-5`.

## 3. SSH key login

**The idea:** a key pair has a public key (the lock) and a private key (the key).
- The machine that **checks** holds the public key
- The machine that **proves** holds the private key

| Login | Public key lives on | Private key lives on |
|---|---|---|
| Laptop to my EC2 server | Server (`~/.ssh/authorized_keys`) | Laptop |
| Server to GitHub | GitHub (Settings, SSH keys) | Server |

Steps I did:
1. `ssh-keygen -t ed25519 -f <file> -C "label"` on my laptop
2. Copied the `.pub` file contents to the server with `echo '...' >> ~/.ssh/authorized_keys`
3. `chmod 700 ~/.ssh` and `chmod 600 ~/.ssh/authorized_keys` because SSH ignores keys if other users can write to these
4. `ssh-keygen -lf` to check the key was valid and to compare fingerprints
5. Connected with `ssh -i <private key> ubuntu@<server IP>`

I proved I was in an SSH session with `echo $SSH_CONNECTION`.

**Rule I remember:** the private key never leaves the machine that created it and never goes in a repo.

## 4. Git and GitHub

Steps:
1. `git init` inside `~/linux-labs` (not my home folder)
2. `git branch -m main` so the branch name matches GitHub
3. `git config --global user.name` and `user.email`
4. `.gitignore` containing `monitor.out` and `app.log`
5. `git add monitor.sh .gitignore`, then `git commit -m "..."`
6. Made an SSH key on the server for GitHub, added the `.pub` to GitHub
7. `~/.ssh/config` with a `Host github.com` block
8. `git remote add origin ...` then `git push -u origin main`

**Why `.gitignore`:** we commit what we write, not what a program produces. `monitor.out` grows every cron run, and real logs can contain sensitive data.

**What the `Host github.com` block does:** it is a rule saved on my machine in `~/.ssh/config`. Whenever I connect to github.com, SSH logs in as user `git` and uses the key file I named. For other hosts it is ignored.

I tested the GitHub connection with `ssh -T git@github.com`. The message "You've successfully authenticated, but GitHub does not provide shell access" is the success message.

## What broke (incident notes)

| What happened | Cause | How I found and fixed it |
|---|---|---|
| `syntax error: unexpected end of file` | Missing closing `fi` | Read the line number in the error, fixed the block |
| Ran `ssh-keygen` and `ssh` on the wrong machine | Did not check the prompt | Prompt `ubuntu@ip-...` is the server, `C:\Users\...>` is the laptop |
| `Load key ... error in libcrypto` | Passed `authorized_keys` to `-i` instead of the private key | `-i` takes the private key file only |
| `ssh-keygen` gave "no such file or directory" on a new laptop | `.ssh` folder did not exist | Created it with `mkdir` |
| `Permission denied (publickey)` | Server had the public key from my other laptop | Compared fingerprints with `ssh-keygen -lf`, added this laptop's key |
| Key was missing a character | Retyped the key by hand | Copy and paste, never retype |
| `~./ssh` typo (twice) | Dot and slash swapped | It is `~/.ssh`, slash first |
| `unsupported option "yesHost"` in `~/.ssh/config` | Extra word on line 5 | `cat -n` showed the exact line |
| `Repository not found` on push | Repo is `linux-lab`, I typed `linux-labs` | Copied the address from GitHub |

## Things to remember

- Exit code `0` means success and non-zero means failure. Cron and CI tools use this to decide what happens next, since they cannot read output the way I do.
- Always check which machine I am typing on by reading the start of the prompt.
- Predict the result before running a test, then compare.
- Absolute paths in anything scheduled.
