# OverTheWire Natas Writeup (Levels 0-9)
---

## Level 0 → Level 1
**Goal:** Find the password for the next level.

The page displays a simple message: "You can find the password for the next level on this page."

### Solution
This level teaches the fundamental habit of checking the HTML source code. Comments and hidden fields often contain sensitive information left by developers.

1.  Open the Developer Tools by right-clicking and selecting **Inspect**.
2.  Navigate to the **Sources** tab.
3.  Look inside the HTML code; the password is hidden in a comment.
<img width="900" height="322" alt="image" src="https://github.com/user-attachments/assets/731a46f2-bbb3-4496-bd73-2c8db212a16f" />


---

## Level 1 → Level 2
**Goal:** Find the password, but right-clicking has been blocked.

### Solution
The website tries to prevent you from viewing the source code by capturing the right-click event with JavaScript. However, this is a client-side restriction and is easily bypassed.

1.  Since right-click is blocked, use the keyboard shortcut `Ctrl+Shift+I` to open the Developer Tools.
2.  Navigate to the source code to find the password for Level 2 in the comments.
   
<img width="900" height="434" alt="image" src="https://github.com/user-attachments/assets/eebef18d-2b11-4f4c-8367-dd0b2d18c96c" />

---

## Level 2 → Level 3
**Goal:** The page says "There is nothing on this page."

### Solution
We need to find hidden files that are not explicitly linked.

1.  Inspect the source code. You will see an image tag pointing to `files/pixel.png`.
<img width="900" height="434" alt="image" src="https://github.com/user-attachments/assets/95c095c7-cb35-44e4-95c2-d1e63a0a7a26" />
3.  This indicates a directory named `/files/` exists. Add `/files/` to the end of the URL.
4.  Navigate to `http://natas2.natas.labs.overthewire.org/files/` in your browser.
5.  Because directory listing is enabled (a server misconfiguration), you can see all files in that folder.
6.  You will find a file named `users.txt`. Open it to get the password.

---

## Level 3 → Level 4
**Goal:** "Nothing found in the source code."

### Solution
If the source code is clean, the next step is checking standard files used by web crawlers.

1.  Check the `robots.txt` file by appending `/robots.txt` to the URL.
2.  The file contains the following rule:
    ```text
    User-agent: *
    Disallow: /s3cr3t/
    ```
3.  This tells Google/search engines *not* to look in the `/s3cr3t/` folder, which reveals that the folder exists.
4.  Navigate to `/s3cr3t/` by adding it to the URL.
5.  Open the `users.txt` file inside to find the Level 4 password.

---

## Level 4 → Level 5
**Goal:** The page states: "Access disallowed. You are visiting from '' while authorized users should come only from 'http://natas5.natas.labs.overthewire.org/'".

### Solution
The server is checking the HTTP `Referer` header to verify where the request is coming from. We can spoof this header using `curl`.

1.  Open your terminal.
2.  Use the following command to send a request with a modified Referer header:
    ```bash
    curl -u natas4:<password> --referer "[http://natas5.natas.labs.overthewire.org/](http://natas5.natas.labs.overthewire.org/)" "[http://natas4.natas.labs.overthewire.org/](http://natas4.natas.labs.overthewire.org/)"
    ```
3.  The response will contain the password for Natas 5.
<img width="900" height="260" alt="image" src="https://github.com/user-attachments/assets/bc9600c6-62d9-493a-81de-5e5812b4b9c2" />

---

## Level 5 → Level 6
**Goal:** "Access disallowed. You are not logged in."

### Solution
Authentication status is often tracked using cookies. If the server relies solely on a client-side cookie without verification, we can tamper with it.

1.  Check the HTTP headers or cookies. Use the command `curl -I -u natas5:<password> "http://natas5.natas.labs.overthewire.org/"` You will see a `Set-Cookie` header where `loggedin=0`.
<img width="759" height="206" alt="image" src="https://github.com/user-attachments/assets/c804e30b-ab02-4cb8-bf23-c7f0e0942f7d" />
3.  Open Developer Tools and go to the **Application** tab, then select **Cookies**.
<img width="900" height="429" alt="image" src="https://github.com/user-attachments/assets/0915362b-6a4d-4628-a93d-5a444ef7b698" />
5.  Double-click the value for `loggedin` and change it from `0` to `1`.
6.  Refresh the page to get the password for Level 6.

---

## Level 6 → Level 7
**Goal:** The page asks for a "secret" to retrieve the password.

### Analysis
Clicking "View sourcecode" reveals the following PHP logic:

```php
include "includes/secret.inc";
if(array_key_exists("submit", $_POST)) {
    if($secret == $_POST['secret']) {
        print "Access granted...";
    }
}
```

The script includes a file at `includes/secret.inc` and checks your input against the variable `$secret` defined in that file.

### Solution
1.  Navigate manually to `http://natas6.natas.labs.overthewire.org/includes/secret.inc`.
2.  View the source of that page to find the secret variable.
3.  Copy the secret string.
4.  Go back to the main page, paste the secret into the input box, and submit to get the Level 7 password.

---

## Level 7 → Level 8
**Goal:** The page has "Home" and "About" links.

### Analysis
Clicking the links changes the URL to `index.php?page=home` or `index.php?page=about`.
If we inject a random string like `jkn`, we get an error: `include(jkn): failed to open stream`. This confirms the code is using the `include()` PHP function on the `page` parameter, leading to a **Local File Inclusion (LFI)** vulnerability.
<img width="900" height="430" alt="image" src="https://github.com/user-attachments/assets/d0b9423e-f2e6-48c9-a20a-22deda9ef7f7" />

### Solution
We need to read the password for the next level, which is stored in `/etc/natas_webpass/natas8`.

1.  Change the URL parameter to traverse up the directory tree (Path Traversal).
2.  Inject the following payload:
    ```
    index.php?page=../../../../../../etc/natas_webpass/natas8
    ```
3.  The page will render the contents of the password file.

---

## Level 8 → Level 9
**Goal:** Input the correct secret.

### Analysis
The source code shows the secret is encoded using a specific sequence:

```php
$encodedSecret = "3d3d516343746d4d6d6c315669563362";

function encodeSecret($secret) {
    return bin2hex(strrev(base64_encode($secret)));
}
```

To find the secret, we need to reverse the operations in the opposite order:
1.  `hex2bin` (Reverse of bin2hex)
2.  `strrev` (Reverse the string again)
3.  `base64_decode` (Reverse of base64_encode)

### Solution
You can run a quick PHP script to decode the secret:

```php
<?php
$encodedSecret = "3d3d516343746d4d6d6c315669563362";
echo base64_decode(strrev(hex2bin($encodedSecret)));
?>
```
1.  Run the code to get the secret.
2.  Submit the secret to get the Natas 9 password.

---

## Level 9 → Level 10
**Goal:** Find words containing a search string.

### Analysis
The source code uses `passthru` to execute a grep command:

```php
passthru("grep -i $key dictionary.txt");
```

The `$key` variable is taken directly from user input without sanitization. This allows for **Command Injection**.

### Solution
We can use the `;` character to terminate the `grep` command and start a new one.

1.  We need to find the password file for Level 10.
<img width="900" height="561" alt="image" src="https://github.com/user-attachments/assets/6dc45427-f619-4d15-85ab-e1381bb915f9" />
3.  Inject a command using `;` followed by `cat` to read the file.
4.  Search query:
    ```bash
    ; cat /etc/natas_webpass/natas10
    ```
5.  The output will display the contents of `dictionary.txt` (from the grep) followed by the contents of the password file.
