# OverTheWire: Bandit

Bandit is a beginner wargame focused on core Linux command-line skills —
file navigation, permissions, text processing, and basic encoding/ciphers.
Each level requires finding a password to log into the next level as a
different user.

This documents the technique and reasoning behind each level — not the
passwords themselves, per OverTheWire's rules against posting spoilers.

## Levels completed

| Level | Concept |
|-------|---------|
| [0 → 1](#level-0--1) | SSH basics, file navigation |
| [1 → 2](#level-1--2) | Filenames starting with a dash |
| [2 → 3](#level-2--3) | Quoting filenames with spaces |
| [3 → 4](#level-3--4) | Hidden files (`ls -la`) |
| [4 → 5](#level-4--5) | File type identification (`file`) |
| [5 → 6](#level-5--6) | `find` with multiple filters |
| [6 → 7](#level-6--7) | `find` across the filesystem, owner/group |
| [7 → 8](#level-7--8) | `grep` pattern searching |
| [8 → 9](#level-8--9) | `sort` + `uniq -c` frequency counting |
| [9 → 10](#level-9--10) | `strings` — extracting text from binary |
| [10 → 11](#level-10--11) | Base64 decoding |
| [11 → 12](#level-11--12) | ROT13 cipher |

---

### Level 0 → 1

**Goal:** Log in via SSH and read the password stored in the home directory.

**Commands used:**
```
ssh bandit0@bandit.labs.overthewire.org -p 2220
ls
cat readme
```

**What each command/flag does:**
- `-p 2220` — Bandit runs SSH on a non-default port, so it has to be specified explicitly.
- `ls` — lists files in the home directory to see what's available.
- `cat readme` — prints the file contents directly to the terminal.

**Why it worked:**
The password was stored in plaintext inside a file called `readme` in the home directory — no permissions issue, no obfuscation, just needed to find and read the file.

**Concept:** `ssh basics`, `file navigation`

---

### Level 1 → 2

**Goal:** Read a file whose name is just a single dash (`-`).

**Commands used:**
```
ls
cat -- ./-
```

**What each command/flag does:**
- `cat ./-` — the `./` prefix tells the shell "this is a path in the current directory," not a command-line flag.

**Why it worked:**
Running `cat -` alone doesn't work as expected because a leading dash is normally interpreted as the start of a command-line option, not a filename. Prefixing it with `./` (or using `--` before it) forces `cat` to treat it as a literal filename instead.

**Concept:** `filenames starting with a dash`, `./ path prefix`

---

### Level 2 → 3

**Goal:** Read a file whose name contains spaces and is wrapped in double dashes: `--spaces in this filename--`.

**Commands used:**
```
cat -- "--spaces in this filename--"
```

**What each command/flag does:**
- Quotes (`"..."`) — keep the spaces inside the filename together as one argument instead of being split into multiple words.
- `--` — tells `cat` "everything after this is a filename, not a flag," so the leading dashes in the name aren't misread as options.

**Why it worked:**
Two problems stacked: the spaces would normally break the filename into separate arguments, and the leading `--` would normally be read as an option flag. Quoting solved the first problem, and `--` solved the second.

**Concept:** `quoting filenames with spaces`, `-- to stop flag parsing`

---

### Level 3 → 4

**Goal:** Find and read a hidden file inside a subdirectory.

**Commands used:**
```
cd inhere
ls -la
cat ...Hiding-From-You
```

**What each command/flag does:**
- `ls -la` — the `-a` flag shows hidden files (anything starting with a dot), which plain `ls` skips; `-l` shows permissions/owner details.

**Why it worked:**
The file `...Hiding-From-You` starts with dots, so it's treated as a hidden file by default and doesn't show up in a normal `ls`. `-a` reveals it.

**Concept:** `hidden files`, `ls -la`

---

### Level 4 → 5

**Goal:** Out of 10 files in a directory, find the one human-readable file and read its password.

**Commands used:**
```
cd inhere
ls -la
file ./*
cat ./-file07
```

**What each command/flag does:**
- `file ./*` — runs the `file` command against every file in the directory at once, identifying each file's actual type regardless of its name.

**Why it worked:**
All 10 files had misleading names (`-file00` through `-file09`) that gave no clue about their content. `file` inspects the actual file content/structure instead of trusting the filename, and identified `-file07` as `ASCII text` — the only human-readable one.

**Concept:** `file type identification`, `file command`

---

### Level 5 → 6

**Goal:** Find one specific file, out of ~20 nested directories each containing 3 similarly-named files, that is human-readable, non-executable, and exactly 1033 bytes.

**Commands used:**
```
find . -type f ! -executable
find . -type f -size 1033c ! -executable
cat <matching file>
```

**What each command/flag does:**
- `-type f` — restrict results to regular files only (not directories).
- `! -executable` — exclude files that have the executable permission bit set.
- `-size 1033c` — match files that are exactly 1033 bytes (`c` = bytes).

**Why it worked:**
A blind `find . -type f ! -executable` returned too many candidates. Adding the exact byte-size filter narrowed it down to a single matching file.

**Concept:** `find with multiple filters`, `-size`, `-executable`

---

### Level 6 → 7

**Goal:** Find one file, somewhere on the entire filesystem, owned by a specific user and group, with a specific file size.

**Commands used:**
```
find / -user <target-user> -group <target-group> -size 33c 2>/dev/null
cat <matching file>
```

**What each command/flag does:**
- `find /` — search starting from the filesystem root, not just the home directory.
- `-user ... -group ...` — filter by file owner and group.
- `2>/dev/null` — redirect error messages (like "permission denied" on folders you can't access) to nowhere, so they don't clutter the output.

**Why it worked:**
The target file wasn't in any predictable location. Searching the whole filesystem with the right owner/group/size filters found it directly, and silencing permission errors kept the output readable.

**Concept:** `find across the whole filesystem`, `filtering by owner/group/size`, `suppressing stderr`

---

### Level 7 → 8

**Goal:** Find the password next to a specific keyword in a large data file.

**Commands used:**
```
grep "<keyword>" data.txt
```

**What each command/flag does:**
- `grep "<keyword>"` — searches the file for lines containing the target keyword and prints those lines.

**Why it worked:**
The file had many lines of "word + value" pairs. Instead of scrolling through manually, `grep` found the one line containing the target keyword directly.

**Concept:** `grep`, `pattern searching in text`

---

### Level 8 → 9

**Goal:** Out of thousands of duplicate lines in a file, find the one line that appears only once.

**Commands used:**
```
sort data.txt | uniq -c | grep -w 1
```

**What each command/flag does:**
- `sort data.txt` — sorts the file so identical lines end up next to each other (required for `uniq` to work correctly).
- `uniq -c` — collapses adjacent duplicate lines into one, prefixed with a count of how many times each appeared.
- `grep -w 1` — filters that output down to only the line(s) with a count of exactly 1 (`-w` matches the whole word "1", avoiding accidental matches like "10").

**Why it worked:**
Every line in the file appeared the same number of times except one, which appeared once. `sort | uniq -c` surfaces the count for every line, and filtering for count `1` isolates the odd one out.

**Concept:** `sort + uniq -c for frequency counting`, `finding unique/anomalous lines`

---

### Level 9 → 10

**Goal:** Extract a human-readable password out of a file containing mostly binary/garbage data.

**Commands used:**
```
strings data.txt | grep "==="
```

**What each command/flag does:**
- `strings data.txt` — scans a binary/mixed file and prints only the sequences that look like readable text, ignoring non-printable binary bytes.
- `grep "==="` — narrows that output down to the line containing a marker, which flagged where the password was.

**Why it worked:**
The password was embedded inside a mostly-binary file. `strings` pulled out only the human-readable fragments, and grepping for a distinguishing marker isolated the right one.

**Concept:** `strings command`, `extracting text from binary data`

---

### Level 10 → 11

**Goal:** Decode a Base64-encoded password.

**Commands used:**
```
cat data.txt | base64 -d
```

**What each command/flag does:**
- `base64 -d` — decodes Base64-encoded input back into its original plaintext form (`-d` = decode).

**Why it worked:**
The file's raw content was Base64 — a text-safe encoding of binary/plain data, not encryption. Piping it into `base64 -d` reversed the encoding and revealed the plaintext password.

**Concept:** `base64 encoding/decoding`

---

### Level 11 → 12

**Goal:** Decode a ROT13-encoded password.

**Commands used:**
```
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

**What each command/flag does:**
- `tr 'A-Za-z' 'N-ZA-Mn-za-m'` — translates each letter to the one 13 positions later in the alphabet (wrapping around), which both encodes and decodes ROT13 since applying it twice returns the original text.

**Why it worked:**
The file was encoded with ROT13, a simple letter-substitution cipher. `tr` performed the character-by-character substitution needed to reverse it back to plaintext.

**Concept:** `ROT13 cipher`, `tr command for character substitution`

---

## Notes

- Level 12 is in progress.
- No passwords are included, per OverTheWire's rules against posting spoilers.
