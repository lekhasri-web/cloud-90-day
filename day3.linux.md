# Day 3: Processes, signals, systemd, journalctl

**Environment:** AWS EC2 (Ubuntu 24.04), real systemd and journald
**Dates:** 3-4 Nov 2026, finished 11 Nov 2026 (spilled over, absorbed per Rule 4)
**Status:** Complete

---

## 1. What I built

A small Python script (`/home/ubuntu/myapp/app.py`) that prints a line every 5 seconds and exits with code 1 if the flag file `/tmp/crash.flag` exists. A systemd unit (`/etc/systemd/system/myapp.service`) runs it as the non-root user `ubuntu` with `Restart=on-failure` and `RestartSec=3`. The service is started and enabled.

## 2. Theory

**PID and PPID**
- PID is a process's own number. PPID is the PID of the process that started it.
- A child's PPID is always its parent's PID. Two `sleep` commands started from the same shell (PID 1324) both had PPID 1324.
- PID 1 is systemd (shown as `/sbin/init` on Ubuntu).

**ps and top**
- `ps` is a snapshot, `top` is a live view. `ps aux` shows every process on the machine.
- STAT: `S` = sleeping (waiting for something, normal), `R` = running or waiting for CPU, `Z` = zombie (finished, parent has not cleaned up).
- `pgrep -a name` is the simplest check that something is running. `ps aux | grep name` always includes the grep line itself, which is not a real match.

**Signals**
- `kill PID` sends SIGTERM (15): a polite request. The process can clean up or ignore it.
- `kill -9 PID` sends SIGKILL: the kernel ends the process. It cannot be caught or ignored. Last resort.
- Bash messages: `Done` = finished by itself, `Terminated` = stopped by SIGTERM, `Killed` = stopped by SIGKILL.
- Exit code 0 = success, anything else = failure. Check with `echo $?`.

**systemd**
- `Loaded: enabled/disabled` = does it start at boot? `Active: active (running)` = is it running right now? They are independent.
- `start` = now. `enable` = at every boot.
- `enable` reads `WantedBy=multi-user.target` from the unit file and creates a link in `/etc/systemd/system/multi-user.target.wants/`. At boot, systemd starts everything in that list.
- States: `inactive (dead)` = not running, `failed` = tried and broke, `activating (auto-restart)` = waiting to retry after a failure.
- After creating or editing a unit file: `sudo systemctl daemon-reload`.
- systemd never guesses. Key names, paths and file names must match exactly.

**journalctl**
- Every journal line has a priority label. Plain `print` output gets info (6), even on stderr and even if the text says ERROR.
- `-p err` shows only priority 3 or lower, so plain printed lines are hidden.
- Starting a line with `<3>` makes journald label it as an error.
- `Ctrl+C` on `journalctl -f` stops the viewer, not the service.

## 3. Commands used

| Goal | Command |
|---|---|
| My shell's PID | `echo $$` |
| PID of last background job | `echo $!` |
| Exit code of last command | `echo $?` |
| All processes | `ps aux` |
| Find by name | `pgrep -a name` |
| Polite stop / force stop | `kill PID` / `kill -9 PID` |
| Service state | `systemctl status myapp --no-pager` |
| Start / stop / enable | `sudo systemctl start myapp` / `stop` / `enable` |
| Re-read unit files | `sudo systemctl daemon-reload` |
| Last 20 log lines | `journalctl -u myapp -n 20 --no-pager` |
| Follow logs live | `journalctl -u myapp -f` |
| Only errors, last 5 min | `journalctl -u myapp -p err --since "5 min ago" --no-pager` |
| Hide the noise | `journalctl -u myapp --no-pager \| grep -v "all good"` |

## 4. Final exercise results

**kill -9 test**
- Old Main PID 16793. I ran `sudo kill -9 16793`.
- About 3 seconds later systemd started a new process, Main PID 17228.
- `Restart=on-failure` with `RestartSec=3` did this.
- Journal: `code=killed, status=9/KILL` -> `Failed with result 'signal'` -> `Scheduled restart job` -> `Started myapp.service`.

**Crash test**
- With `/tmp/crash.flag` present, the script printed the error and exited with 1.
- Journal: `code=exited, status=1/FAILURE`.
- While the flag existed, the service crashed and restarted every ~3 seconds (restart counter reached 14, new PID each time). After the flag was removed it stayed up.
- `code=exited` = the program ended itself. `code=killed` = a signal from outside ended it.

**Finding the error with -p err**
- `journalctl -u myapp -p err --since "5 min ago" --no-pager` first returned `-- No entries --`.
- After changing the error print to start with `<3>`, it showed the ERROR lines.

**Enable**
```
sudo systemctl enable myapp
Created symlink /etc/systemd/system/multi-user.target.wants/myapp.service -> /etc/systemd/system/myapp.service.
```
`Loaded: ... enabled` and `Active: active (running)` confirmed.

## 5. Incident log

| # | Symptom | Diagnosis | Root cause | Fix |
|---|---|---|---|---|
| 1 | Flag file present but script kept running | `grep FLAG app.py` showed `"tmp/crash.flag"` | Missing leading `/`, so a relative path (`~/myapp/tmp/crash.flag`) | Changed to `"/tmp/crash.flag"` |
| 2 | Unit file would not run the script | Compared the file with `which python3` | `exec=` instead of `ExecStart=`, and `/user/bin/python3` instead of `/usr/bin/python3` | Corrected key name and path, verified with `grep ExecStart` |
| 3 | `activating (auto-restart)`, `status=2`, restart counter above 75 | `systemctl stop myapp`, then `journalctl -u myapp -n 15 --no-pager`: `can't open file ... No such file or directory`; `ls -l` showed only `app.py.save` | `app.py` did not exist. Only nano's leftover `app.py.save` was there | Recreated `app.py`, tested by hand, then started the service |
| 4 | `journalctl -p err` showed `-- No entries --` during a real crash | Reasoned from priority labels | stderr output is stored at info (6), not error (3) | Prefixed the error print with `<3>` |

Smaller slips: typos in command names (`jounralctl`, `maypp`) gave `command not found` and `Unit could not be found`. Both mean check the spelling.

## 6. What I check first when a service fails

1. `systemctl status myapp --no-pager`: tells me the state and how it ended (exit code or signal).
2. `journalctl -u myapp -n 20 --no-pager`: tells me why, from the program's own error message and the restart history.

If it is in a restart loop, run `sudo systemctl stop myapp` first so the log stops filling up. Use `-p err` later to narrow a long log, not as the first look.

## 7. Mistake pattern to watch

My slips today were mostly one character: a missing `/`, `exec` vs `ExecStart`, `user` vs `usr`. When something fails silently, compare it letter by letter with a known-good version.

## 8. Checklist

- [x] Processes, ps/top, signals
- [x] systemctl and unit file
- [x] journalctl (`-u`, `-f`, `--since`, `-p err`)
- [x] Service runs as non-root and restarts after `kill -9`
- [x] Crash found in the journal
- [x] `enable` confirmed
- [ ] Day 7 closed-book recall: retell the kill -9 restart from memory
