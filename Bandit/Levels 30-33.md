# OverTheWire Bandit Writeup (Levels 30–33)

## Level 30 → Level 31

You are given another internal Git repository. After cloning it, the working tree looks empty or unhelpful, but Git objects can also be referenced through tags.

```bash
git clone ssh://bandit30-git@bandit.labs.overthewire.org:2220/home/bandit30-git/repo
cd repo
ls
cat README.md
```

The README contains no useful data. Next, enumerate tags:

```bash
git tag
```

Tags are named references to specific Git objects (usually commits). Display the tagged object:

```bash
git show <tagname>
```

`git show` prints the object the tag points to, including commit message and file contents if applicable. Tags are often overlooked during repository audits, but they can reference hidden or older data.

---

## Level 31 → Level 32

The repository instructions require you to push a specific file to the remote.

```bash
git clone ssh://bandit31-git@bandit.labs.overthewire.org:2220/home/bandit31-git/repo
cd repo
cat README.md
```

Follow the requirements exactly: filename, content, and branch.

Create the file:

```bash
echo "May I come in?" > key.txt
```

If the repository uses ignore rules, force‑add the file:

```bash
git add -f key.txt
```

Commit and push:

```bash
git commit -m "add key"
git push origin master
```

The remote repository runs a server‑side validation hook. Even if the push is rejected afterward, the hook output reveals the next password. This demonstrates how Git hooks can enforce rules and also return automated responses.

---

## Level 32 → Level 33

You are dropped into a restricted uppercase shell where commands are converted to uppercase before execution.

Try running a command and observe the failure. Then print the current shell path using a special parameter:

```
$0
```

`$0` expands to the current shell executable and launches it directly, bypassing the uppercase filter. From there, start a normal editor:

```bash
cat /etc/bandit_pass/bandit33
```

## Level 33

This is the final level. After logging in, you only need to read the message file.

```bash
ls
cat README.txt
```

The file confirms completion of the Bandit wargame. No further exploitation is required — this level serves as a completion checkpoint.
