# OverTheWire Bandit Writeup — Part 1 (Levels 0–9)

This writeup covers the first part of the OverTheWire Bandit wargame (Levels 0–9). The goal is educational: instead of only listing commands, each step explains *why* a command is used and what Linux concept it teaches.

All passwords are intentionally omitted. Only the method and reasoning are shown so you can learn and reproduce the results yourself.

---

## Level 0 → Level 1

**Goal:** Log into the Bandit server using SSH and find the password stored in a file called `readme`.

### Connect to the server

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

**Why:**

* `ssh` is used for secure remote login.
* `bandit0@...` specifies the username and host.
* `-p 2220` sets a custom port (default SSH port is 22, but Bandit uses 2220).

### Read the file

```bash
ls
cat readme
```

**Why:**

* `ls` lists files in the directory so we can confirm `readme` exists.
* `cat` prints file contents to the terminal. Since the password is stored directly in the file, this is the simplest method.

---

## Level 1 → Level 2

**Goal:** The password is stored in a file named `-`.

### Read the file named `-`

```bash
cat < -
```

**Why:**

* A filename of `-` is tricky because many commands interpret it as standard input.
* Using input redirection (`< -`) forces the shell to treat `-` as a filename instead of stdin.
* This level teaches that special characters and reserved names can change command behavior.

---

## Level 2 → Level 3

**Goal:** The password is in a file called `--spaces in this filename--`.

### Read a filename with spaces

```bash
cat "--spaces in this filename--"
```

**Why:**

* Spaces normally separate command arguments.
* Quoting the filename tells the shell to treat it as one argument.
* This level teaches proper quoting and escaping.

---

## Level 3 → Level 4

**Goal:** The password is in a hidden file inside the `inhere` directory.

### Find hidden files

```bash
cd inhere
ls -la
cat ...Hiding-From-You
```

**Why:**

* `cd inhere` changes into the target directory.
* `ls -la` lists **all** files, including hidden ones (those starting with `.`).
* Hidden files are not shown by default with plain `ls`.
* This level teaches directory navigation and hidden file discovery.

---

## Level 4 → Level 5

**Goal:** The password is in the only human-readable file in the directory.

### Detect readable file type

```bash
cd inhere
file ./-file*
cat ./-file07
```

**Why:**

* Many files are present and most are binary.
* The `file` command inspects file contents and reports their type.
* `./-file*` uses a wildcard and `./` to avoid problems with filenames starting with `-`.
* Once the ASCII text file is identified, `cat` is used to read it.

---

## Level 5 → Level 6

**Goal:** Find a file under `inhere` that is:

* human-readable
* 1033 bytes
* not executable

### Use find with filters

```bash
cd inhere
find -type f -readable ! -executable -size 1033c
cat ./maybehere07/.file2
```

**Why:**

* `find` searches directories recursively.
* `-type f` limits results to files.
* `-readable` ensures we can read it.
* `! -executable` excludes executable files.
* `-size 1033c` filters by exact byte size.

This level teaches structured searching using metadata instead of guessing.

---

## Level 6 → Level 7

**Goal:** Find a file anywhere on the server with:

* owner = bandit7
* group = bandit6
* size = 33 bytes

### Search from root and suppress errors

```bash
find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
cat /var/lib/dpkg/info/bandit7.password
```

**Why:**

* Searching from `/` scans the entire filesystem.
* Ownership filters narrow results quickly.
* Many directories are not accessible, causing permission errors.
* `2>/dev/null` hides error messages by redirecting stderr.

---

## Level 7 → Level 8

**Goal:** The password is in `data.txt` next to the word `millionth`.

### Search inside a file

```bash
grep "millionth" data.txt
```

**Why:**

* `grep` searches for matching text patterns inside files.
* Faster and more precise than manually reading a large file.

---

## Level 8 → Level 9

**Goal:** The password is the only line that appears once.

### Sort and count unique lines

```bash
sort data.txt | uniq -u
```

**Why:**

* `uniq` only detects duplicates if identical lines are adjacent.
* `sort` groups identical lines together first.
* `uniq -u` prints only lines that appear once.

---

## Level 9 → Level 10

**Goal:** The password is in one of the human-readable strings in a binary file, preceded by several `=` characters.

### Extract readable strings

```bash
strings data.txt | grep "==="
```

**Why:**

* `strings` extracts printable text from binary data.
* Piping to `grep` filters only lines containing the marker pattern.
