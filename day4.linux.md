# Day 4: Linux Text Tools (redirection, pipes, grep, sed, awk)

**Date:** 12 Nov 2026
**Environment:** AWS EC2 `linux-lab` (Ubuntu 24.04), practice folder `~/textlab`
**Goal:** Read and filter logs comfortably from the command line.

The pattern behind most log work: **filter (grep) -> extract (awk/cut) -> normalize (sed) -> sort -> count (uniq -c)**.

---

## 1. Redirection

| Symbol | Meaning |
|---|---|
| `>` | stdout to a file, **empties the file first** |
| `>>` | stdout appended to the end of a file |
| `2>` | stderr to a file |
| `> f 2>&1` | stdout and stderr to the same file |
| `2> /dev/null` | discard errors |

Every command has three streams: stdin (0), stdout (1), stderr (2).

**Order matters:** `> out.txt 2>&1` sends both to the file. `2>&1 > out.txt` sends errors to the screen, because stderr copied stdout's destination *before* stdout moved.

**Rule I learned the hard way:** before `>` is what a command reads, after `>` is where it writes. Bash opens (and empties) the file after `>` *before* the command runs.

---

## 2. Pipes and helpers

- `|` sends the stdout of the left command into the stdin of the right one. stderr does not travel through a pipe.
- `sort` groups identical lines. `uniq -c` only counts **adjacent** duplicates, so always `sort | uniq -c`.
- `sort -rn` sorts numerically, highest first.
- `head -n N` / `tail -n N` take the first / last N lines. `wc -l` counts lines.
- `tee file` saves the stream to a file *and* keeps passing it on. Its position in the pipe decides what gets saved:
  - `... | tee f | head -n 1` saves everything.
  - `... | head -n 1 | tee f` saves only one line.

```bash
sort fruits.txt | uniq -c | sort -rn
```

---

## 3. grep (find lines)

| Flag | Use |
|---|---|
| `-i` | ignore case |
| `-v` | invert (lines that do NOT match) |
| `-n` | show line numbers |
| `-c` | count matching lines |
| `-A N` / `-B N` | N lines after / before a match |
| `-E "A\|B"` | A **or** B (plain grep treats `\|` as a literal) |
| `-r` | search a whole folder |

**Gotcha:** grep is case-sensitive. `grep ERROR` missed the lowercase `error disk almost full` line, which was the one that mattered. Use `-i` when hunting.

```bash
grep -n "db connection" app.log
grep -E "ERROR|WARN" app.log
```

---

## 4. sed (edit text)

- `s/old/new/` replaces the **first** match on each line. `s/old/new/g` replaces **every** match.
- A substitute needs three slashes, and the whole expression goes inside one pair of single quotes.
- `sed -n '4p'` prints only line 4. `sed '/INFO/d'` deletes matching lines from the output.
- **Plain sed never changes the file.** It prints an edited copy.

Safe ways to save:

| Method | Effect |
|---|---|
| `sed '...' app.log > new.log` | new file, original untouched (safest) |
| `sed -i.bak '...' file` | edits in place, **original saved as `file.bak`** |
| `sed -i '...' file` | edits in place, **no undo** (risky) |

`diff file.bak file` shows the before (`<`) and after (`>`).

Masking example (hides every username without naming them):

```bash
sed 's/user=[^ ]*/user=XXX/' app.log > masked.log
```

`[^ ]*` means "any run of characters that are not a space".

---

## 5. awk (work with columns)

- `$1`, `$2` ... are fields, `$0` is the whole line, `$NF` (uppercase) is the last field.
- `NR` is the line number, `NF` is the field count.
- `print $2, $3` puts a space between fields. Without the comma they are glued together.
- awk treats a run of spaces as one separator. `-F'='` changes the separator.
- Conditions: `$3 == "ERROR"` matches only field 3. `/time=/` matches text anywhere on the line.
- Text to compare must be in **double quotes**. An unquoted `ERROR` is an empty variable.
- The whole awk program goes in **single quotes**, otherwise bash expands `$3` first.

```bash
awk '$3 == "ERROR" {print $2}' app.log > error_times.txt
awk -F'=' '/time=/ {sum+=$NF} END{print sum}' app.log     # 1350
```

**Filter first, then calculate.** Without `/time=/` the sum was 13482, because awk also added the port (8080) and two years (2026 each), with no error message.

---

## 6. cut vs awk

`cut -d' ' -f4` treats **every single space** as a separator, so the double space after `INFO` created an empty field and shifted the columns. `awk '{print $4}'` treats runs of spaces as one separator. On irregular spacing, use awk. `cut` suits fixed formats such as CSV.

---

## 7. Real logs with journalctl

```bash
journalctl --no-pager | wc -l                       # 145,517 lines
journalctl --no-pager -p err | wc -l                # 61 error lines
journalctl --no-pager -p err | awk '{print $5}' | sed 's/\[.*//' | sort | uniq -c | sort -rn | head -n 5
```

Result: `sshd` is the top error source (45), from internet bots hitting port 22.

**Why `sed` goes before `sort`:** `sshd[1960]:` and `sshd[17759]:` count as different values until the PID is stripped. Normalize first, then sort, then count.

---

## Final exercise results

| # | Task | Result |
|---|---|---|
| 1 | Count `ERROR` lines | 3 |
| 2 | Count `error` in any case | 4 |
| 3 | Top 2 log levels | INFO (4), ERROR (3) |
| 4 | ERROR timestamps to a file | `error_times.txt` |
| 5 | Mask users, prove original unchanged | `alice` count: 2 in `app.log`, 0 in `masked.log` |
| 6 | Sum of `time=` values | 1350 |
| 7 | Journal errors, program names without PIDs | 61 lines in, 61 lines out |

Closed-book check at the end of the session: 3 out of 3 correct.

---

## Mistakes and what they taught me

1. **Wiped `app.log` with a stray `>`.** I wrote `sed 's/ERROR/CRITICAL/' > app.log > app_edited.log`. Bash emptied `app.log` before sed ran. Lesson: never put a file I want to keep right after `>`.
2. **Commands hanging with no output** (`grep $`, `awk '{print $2}'`). No filename meant the command waited on stdin. Fix: Ctrl+C, then add the file.
3. **A `>` prompt appearing.** An unclosed single quote. Fix: Ctrl+C and retype with matching quotes.
4. **Unquoted `$3` in awk.** Bash replaced `$3` with an empty value, so awk got `{}` and printed nothing.
5. **`ERROR` vs `"ERROR"` in awk.** The unquoted word was an empty variable, so the condition never matched and the output file was empty.
6. **`>_ file` typo** created a stray file named `_`. Removed with `rm -- _`.
7. **Pasting multi-line blocks into EC2 Instance Connect** dropped or duplicated lines. Fix: paste short blocks and verify with `wc -l` and `cat`.
8. **Wrong folder.** A relative filename only works from the folder that contains it. Check `pwd` first.
9. **`l` (letter) vs `1` (digit)** in `head -n l`.

---

## Cheat sheet: commands I'll reuse

```bash
grep -i "error" file                   # case-insensitive search
grep -c ERROR file                     # count matching lines
awk '{print $3}' file | sort | uniq -c | sort -rn      # count a column
sed 's/old/new/g' file > new.log       # safe replace into a new file
awk -F'=' '/time=/ {sum+=$NF} END{print sum}' file     # filtered sum
journalctl --no-pager -p err | tail -n 20              # recent errors
```

---

## Next

- Day 5: Linux (next topic in the tracker)
- Day 7: closed-book recall of Week 1
