# Day 7: Linux closed-book check, review and Bandit

**Date:** Day 7 (Sunday, Week 1) | **Hours:** 3

## Built
- `count_errors.sh`: takes a log file as an argument and prints the count of ERROR lines.
  - No argument: usage message, exit 1
  - File does not exist: error message, exit 2
  - Success: exit 0
- Tested against a 10-line `app.log`. Predicted 3 ERROR lines before running; the script reported 3.

## Bandit (levels 0-10)
- Learned SSH (`ssh user@host -p 2220`)
- Handling files named `-` and files with spaces in the name
- `find` command
- `uniq -u`

## Broke / fixed
- First run of `./count_errors.sh app.log` gave `Permission denied`. A new file has no execute bit. Fixed with `chmod +x`.

## Still shaky (from flashcards)
- `cat -` waits on stdin. A path (`./-` or `~/-`) reads the file; quotes do not help.
- `~` already means `/home/<user>`, so do not repeat the username after it.
- Read `ls -l` as: file type, then owner, group, others.
- `chmod +x` adds to existing permissions; `chmod 755` sets them exactly (755 = rwxr-xr-x).

## Next
- Redo the closed-book script with no hints on the Week 2 Sunday buffer.
- Add to Recall: `df`/`du`, `find`, hard vs soft links, `apt`, boot process, server hardening.
