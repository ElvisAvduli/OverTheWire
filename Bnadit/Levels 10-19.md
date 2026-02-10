# OverTheWire Bandit Writeup (Levels 10–19)

---

## Level 10 → Level 11

**Level Goal:** The password for the next level is stored in `data.txt`, which contains base64 encoded data.

### Command

```bash
base64 -d data.txt
```

### Why this works

* Base64 is an encoding scheme that represents binary data using printable ASCII characters.
* Because it is encoding and not encryption, no key is required — only decoding.
* The `base64` utility handles both encoding and decoding.
* The `-d` flag switches the tool into decode mode and prints the original content.

---

## Level 11 → Level 12

**Level Goal:** The password is stored in `data.txt`, where all letters have been rotated by 13 positions (ROT13).

### Command

```bash
cat data.txt | tr A-Za-z N-ZA-Mn-za-m
```

### Why this works

* ROT13 is a substitution cipher that shifts each letter by 13 places.
* The `tr` command translates characters from one set into another.
* The mapping `A-Za-z` → `N-ZA-Mn-za-m` performs the exact rotation for uppercase and lowercase letters.
* A pipe is used so the output of `cat` becomes the input of `tr`.

---

## Level 12 → Level 13

**Level Goal:** `data.txt` is a hexdump of a file that has been compressed multiple times. You must reconstruct it step by step.

### Prepare a safe workspace

```bash
workdir=$(mktemp -d)
cd "$workdir"
cp ~/data.txt .
```

### Convert hexdump back to binary

```bash
xxd -r data.txt > stage.bin
```

### Repeatedly identify and extract

```bash
file stage.bin
```

Then apply the correct tool depending on the detected type:

```bash
gunzip stage.bin
bzip2 -d stage.bin
tar -xf stage.bin
```

Rename files when necessary so tools recognize the format.

### Why this works

* A hexdump is a textual representation of binary data.
* `xxd -r` reverses the dump into the original binary.
* The `file` command checks real content type instead of relying on file extensions.
* Compression layers must be removed one at a time in the correct order.
* Working in `/tmp` avoids clutter and permission issues.

---

## Level 13 → Level 14

**Level Goal:** You are given an SSH private key and must use it to log in as the next user.

### Commands
From your terminal: 
```bash
scp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private .
chmod 600 sshkey.private
ssh -i sshkey.private bandit14@localhost -p 2220
```

### Why this works

* SSH supports key-based authentication instead of passwords.
* The `-i` option specifies which private key to use.
* SSH refuses keys that are readable by others, so permissions must be restricted.
* `chmod 600` ensures only the owner can read the key file.

---

## Level 14 → Level 15

**Level Goal:** Submit the current password to port 30000 on localhost to receive the next password.

### Command

```bash
nc localhost 30000
```

### Why this works

* `nc` (netcat) is a simple TCP client for manual connections.
* It lets you interact directly with listening services.
* After connecting, you manually paste the password and read the response.
* This demonstrates basic client/server socket interaction.

---

## Level 15 → Level 16

**Level Goal:** Submit the password to port 30001 using SSL/TLS.

### Command

```bash
openssl s_client -connect localhost:30001
```

### Why this works

* This service requires encrypted TLS instead of plain TCP.
* `openssl s_client` is a diagnostic TLS client.
* It establishes an encrypted session and allows manual input.
* Plain tools like `nc` or `telnet` fail here because they do not negotiate TLS.

---

## Level 16 → Level 17

**Level Goal:** Find which port between 31000–32000 provides the correct SSL service and returns credentials.

### Scan the port range

```bash
nmap -p 31000-32000 localhost
```

### Test SSL-enabled ports

```bash
openssl s_client -connect localhost:<port>
```

### Why this works

* `nmap` discovers which ports are open.
* Only some open ports speak SSL/TLS.
* Testing each candidate with `openssl s_client` reveals which one returns useful data.
* The correct service returns an SSH private key for the next login.

---

## Level 17 → Level 18

**Level Goal:** Two files differ by exactly one line. That line is the new password.

### Command

```bash
diff passwords.old passwords.new
```

### Why this works

* `diff` compares files line by line.
* It highlights additions and removals.
* Since only one line changed, the output directly reveals the target value.
* This is more reliable than manual inspection for large files.

---

## Level 18 → Level 19

**Level Goal:** `.bashrc` logs you out immediately on SSH login. You must bypass it and read `readme`.

### Command

```bash
ssh bandit18@bandit.labs.overthewire.org -p 2220 -t /bin/sh
```

Then run:

```bash
cat readme
```

### Why this works

* The default login shell executes `.bashrc`, which forces logout.
* Supplying a command (`/bin/sh`) tells SSH to run that shell directly.
* This bypasses the normal interactive bash startup.
* The `-t` flag forces a pseudo-terminal so the shell behaves interactively.

---

## Level 19 → Level 20

**Level Goal:** Use a setuid helper binary to read a protected password file.

### Command

```bash
./bandit20-do cat /etc/bandit_pass/bandit20
```

### Why this works

* The helper binary has the setuid bit set.
* Setuid programs run with the file owner’s privileges.
* This allows the command to execute as bandit20 instead of the current user.
* It provides controlled privilege escalation for a single command.
