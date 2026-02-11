# OverTheWire Bandit Writeup — Levels 20 → 29 

---

## Level 20 → Level 21

A setuid program named `suconnect` connects to a port you specify, reads one line from that connection, compares it to the current level password, and if it matches, sends back the next password.

You need to simulate a small server locally that sends the password when `suconnect` connects.

Open two terminals.

**Terminal A — create a listening service:**

```bash
nc -lp 31337 < /etc/bandit_pass/bandit20
```

This starts netcat in listen mode on port 31337. The `<` redirection feeds the password file into the connection, so any client that connects immediately receives that line.

**Terminal B — trigger the setuid client:**

```bash
./suconnect 31337
```

The binary connects to localhost:31337, reads the password you served, validates it, and returns the next password. This level demonstrates that you can emulate network services locally using simple tools and file redirection.

---

## Level 21 → Level 22

A cron job runs automatically and copies the next password somewhere. Your job is to discover where.

Start by inspecting cron configuration files:

```bash
ls /etc/cron.d/
cat /etc/cron.d/cronjob_bandit22
```

Cron entries show which user runs which script and how often. The entry points to a script path — open it:

```bash
cat /usr/bin/cronjob_bandit22.sh
```

Read the script line by line. It shows the exact destination file in `/tmp`. Once you know the path, simply read it with `cat`. The key skill here is tracing execution from scheduler → config → script → output file.

---

## Level 22 → Level 23

Another cron job writes the password to `/tmp`, but the filename is dynamically generated from a hash.

Open the script:

```bash
cat /usr/bin/cronjob_bandit23.sh
```

You will see a command that hashes a predictable string using `md5sum`. Reproduce that same string exactly and hash it yourself:

```bash
echo "I am user bandit23" | md5sum
```

The hash output becomes the filename. Use it directly under `/tmp/` to retrieve the password. This level teaches that deterministic hashes can be reversed when the input pattern is known.

---

## Level 23 → Level 24

A cron job executes every script placed in `/var/spool/bandit24` as user bandit24, then deletes it.

Create your own script that copies the password to a place you control.

```bash
mkdir /tmp/job
cd /tmp/job
nano grab.sh
```

Script contents:

```bash
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/job/result.txt
```

Make it executable and world‑readable so cron can run it:

```bash
chmod 777 grab.sh
cp grab.sh /var/spool/bandit24/
```

After about a minute the cron job runs it automatically. Then read the output file you created. This demonstrates how scheduled privileged execution can be abused when writable execution directories exist.

---

## Level 24 → Level 25

A service expects the correct password plus a 4‑digit PIN. Since the PIN space is only 0000–9999, you can try all combinations automatically.

Generate all PINs and pipe them into the service:

```bash
for i in {0000..9999}; do
  echo "<password> $i"
done | nc localhost 30002
```

Each line is sent as one attempt. Watch the output — one response will differ and include the next password. This shows how small keyspaces are vulnerable to brute force with simple shell loops.

---

## Level 25 → Level 26

Logging in as bandit26 immediately disconnects you because the login shell runs a custom program instead of bash.

Check which program is used:

```bash
grep bandit26 /etc/passwd
cat /usr/bin/showtext
```

It runs `more` on a text file and exits. If the text fits on one screen, `more` exits instantly. Shrink your terminal height so paging is required, then log in again with the provided SSH key. When `more` pauses at `--More--`, press:

```
v
```

This opens the file in `vi`. From inside vi you can open any readable file, including the password file.

---

## Level 26 → Level 27

Inside vi, you can execute shell commands.

Spawn a shell:

```bash
:set shell=/bin/bash
:!bash
```

Now you have a shell with bandit26 permissions. List files and find the setuid helper binary. Execute it to read the next password file on behalf of bandit27.

---

## Level 27 → Level 28

You are given access to a local Git repository over SSH. Clone it into a temporary directory.

```bash
mkdir /tmp/repo
cd /tmp/repo
git clone ssh://bandit27-git@localhost/home/bandit27-git/repo
cd repo
```

Inspect repository contents with `ls` and `cat`. The password is stored in a tracked file. This level introduces Git as a storage medium that may contain secrets.

---

## Level 28 → Level 29

The password was removed from the latest commit, but Git keeps full history.

View commit history:

```bash
git log
```

Checkout an earlier commit:

```bash
git checkout <commit>
cat README.md
```

Older snapshots still contain the secret. This demonstrates that deleting a secret in a later commit does not remove it from repository history.

---

## Level 29 → Level 30

The repository uses multiple branches. The default branch hides the password.

List branches:

```bash
git branch -a
```

Switch to another branch such as `dev`:

```bash
git checkout dev
cat README.md
```

Different branches can contain entirely different file versions. Always enumerate branches and tags when auditing a repository.
