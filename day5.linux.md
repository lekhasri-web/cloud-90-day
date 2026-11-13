# Day 4: Bash Scripting

Did this on my EC2 instance (`linux-lab`, Ubuntu 24.04). Goal was to write a log-summary script that takes arguments.

## What I learned

**Variables and quotes**
- `name="Lekha"`, no spaces around `=`, otherwise bash thinks `name` is a command.
- Single quotes print whatever is between them as it is. Double quotes will identify the variable and print its value.
- `${name}` tells bash where the variable name ends. `$name_Sri` printed empty because bash looked for a variable called `name_Sri`, which I never created.

**Arguments**
- `$0` is the script name, `$1` and `$2` are the arguments, `$#` is how many arguments were passed.
- `${2:-ERROR}` means use the second argument, and if it is not given, use `ERROR`.

**Exit codes**
- `0` is success, anything else is failure. `ls /nonexistent` gave `2`, `ls /tmp` gave `0`.
- `echo $?` shows the exit code of the last command.
- Whenever bash encounters `exit` it stops running further lines and ends the program with the exit code passed to it.
- In my script: `1` for no argument, `2` for file not found, `0` for success.

**if**
- `if [[ condition ]]; then ... fi`
- Space is needed after `[[` and before `]]`.
- `-f` checks it is a file, `-lt` is less than, `!` is NOT.

**Loops**
- `for x in a b c; do ... done` runs once for each item. Items are separated by spaces, not commas.
- `while read -r line; do ... done < file` reads the file line by line. When the file runs out of lines, `read` fails and the loop stops.

**Functions**
- `usage() { ... }`, then call it by typing `usage`.
- `return 1` just stops running lines inside that function, whereas `exit 1` stops running all lines in the file.

**Log commands**
- `grep -c "WARN" file` counts the lines that match.
- `grep "ERROR" file | tail -n 3` gives the last 3 matching lines.

## Final exercise: `logsum.sh`

Takes a log file and an optional keyword (default `ERROR`). Prints how many INFO, WARN and ERROR lines there are, then the last 3 lines with the keyword.

| Test | Result | Exit code |
|---|---|---|
| `./logsum.sh` | usage message | 1 |
| `./logsum.sh /nonexistent` | error message | 2 |
| `./logsum.sh test.log` | INFO 2, WARN 1, ERROR 4 and last 3 ERROR lines | 0 |
| `./logsum.sh test.log WARN` | same counts, last line is the WARN line | 0 |

Script is in this folder: `logsum.sh`

## Mistakes I made (incident log)

| Error | Why it happened | Fix |
|---|---|---|
| `[[0: command not found` | no space after `[[` | `[[ $# -lt 1 ]]` |
| `syntax error near ;` | no space before `]]` | `1 ]]` |
| `unexpected EOF` | a double quote was not closed | close every quote |
| `echo:: command not found` | typed a colon after `echo` | `echo "Read: $line"` |
| `.checkfile.sh: command not found` | forgot the `/` | `./checkfile.sh` |
| `trail: command not found` | typo | `tail` |
| `test.log is not regular file` | I was in the wrong folder and the file was not created yet | checked with `pwd` and `ls` |

Other bugs in `logsum.sh`: missing `()` after `usage`, `lt` without the dash, used `$0` instead of `$logfile`, commas in the `for` list, single quotes stopped the variables from working, and forgot `done`.

## What I will remember
- Most of my bugs were small typos like spaces, quotes and brackets. The error message shows the line to check.
- Always check `pwd` when a file is "not found".
- Next: run a script as a systemd service and read its logs with `journalctl`.
