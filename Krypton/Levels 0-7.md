# Krypton – OverTheWire Writeup

## Overview

Krypton is a cryptography-focused wargame from OverTheWire that introduces classical cryptographic techniques and demonstrates why they are insecure. The challenges progressively cover:

- Base64 encoding
- Caesar (ROT) cipher
- Monoalphabetic substitution & frequency analysis
- Vigenère cipher (known key length)
- Vigenère cipher (unknown key length)
- Stream cipher weaknesses

---

## Level 0 

**Given:** `S1JZUFRPTklTR1JFQVQ=`

### Concept
Base64 is an **encoding scheme, not encryption**. It converts binary data into ASCII using 64 symbols. It is fully reversible and requires no key.

### Solution

```bash
echo "S1JZUFRPTklTR1JFQVQ=" | base64 -d
```

**Output:** `KRYPTONISGREAT`

> **Password for krypton1:** `KRYPTONISGREAT`

---

## Level 1 

**Ciphertext:** `YRIRY GJB CNFFJBEQ EBGGRA`

### Concept
A Caesar cipher shifts each letter by a fixed number of positions in the alphabet. With only 25 possible shifts, brute force is trivial.

### Solution

```python
import string

cipher = "YRIRY GJB CNFFJBEQ EBGGRA"
alphabet = string.ascii_uppercase

for shift in range(26):
    result = ""
    for c in cipher:
        if c in alphabet:
            result += alphabet[(alphabet.index(c) - shift) % 26]
        else:
            result += c
    print(shift, result)
```

The correct shift is **13** (ROT13), revealing: `LEVEL TWO PASSWORD ROTTEN`

> **Password for krypton2:** `ROTTEN`

---

## Level 2 

**Ciphertext:** `OMQEMDUEQMEK`

### Concept
We have access to an encryption binary. By encrypting chosen plaintext and observing the output, we can determine the shift directly — one comparison is enough.

Brute forcing all shifts reveals the plaintext.

> **Password for krypton3:** `CAESARISEASY`

---

## Level 3 

Multiple ciphertext files are provided, all encrypted with the same substitution key.

### Concept
Each plaintext letter maps to exactly one ciphertext letter (arbitrary mapping, unlike Caesar). This is broken via **frequency analysis** — English has predictable letter frequencies:

- **Most common:** E, T, A, O, I, N
- **Least common:** Q, Z, J

### Process

```bash
# Combine all ciphertext files and count letter frequencies
cat * | tr -d '\n' | fold -w1 | sort | uniq -c | sort -nr
```

Map the most frequent ciphertext letters to the most frequent English letters, then refine until readable English appears.

**Decrypted:** `WELL DONE THE LEVEL FOUR PASSWORD IS BRUTE`

> **Password for krypton4:** `BRUTE`

---

## Level 4 
**Key length given:** 6

### Concept

```
C = (P + K) mod 26
P = (C - K + 26) mod 26
```

A Vigenère cipher applies a different Caesar shift to each character, cycling through the key. If the key length is known, each position can be attacked independently as a simple Caesar cipher using frequency analysis.

### Key Recovery Script (Bash)

```bash
#!/usr/bin/env bash

CIPHER="<FULL CIPHER TEXT HERE>"
KEY_LEN=6

declare -A ENG_FREQ
ENG_FREQ=([A]=820 [B]=150 [C]=280 [D]=430 [E]=1270 [F]=220 [G]=200
           [H]=610 [I]=700 [J]=15  [K]=77  [L]=400 [M]=240 [N]=670
           [O]=750 [P]=190 [Q]=10  [R]=600 [S]=630 [T]=910 [U]=280
           [V]=100 [W]=240 [X]=15  [Y]=200 [Z]=7)

clean_cipher() {
  echo "$1" | tr -cd 'A-Za-z' | tr 'a-z' 'A-Z'
}

get_subsequence() {
  local txt="$1" start="$2" step="$3"
  local result=""
  for (( i=start; i<${#txt}; i+=step )); do
    result+="${txt:$i:1}"
  done
  echo "$result"
}

score_english() {
  local txt="$1"
  local score=0
  declare -A counts
  for (( i=0; i<${#txt}; i++ )); do
    c="${txt:$i:1}"
    counts[$c]=$(( ${counts[$c]:-0} + 1 ))
  done
  for letter in "${!counts[@]}"; do
    freq=${ENG_FREQ[$letter]:-0}
    score=$(( score + freq * ${counts[$letter]} ))
  done
  echo "$score"
}

caesar_decrypt() {
  local txt="$1" shift="$2"
  local result=""
  for (( i=0; i<${#txt}; i++ )); do
    c="${txt:$i:1}"
    o=$(printf '%d' "'$c")
    p=$(( (o - 65 - shift + 26) % 26 ))
    result+=$(printf "$(printf '%03o' $((p + 65)))")
  done
  echo "$result"
}

CLEAN=$(clean_cipher "$CIPHER")
KEY=""

for (( pos=0; pos<KEY_LEN; pos++ )); do
  subseq=$(get_subsequence "$CLEAN" "$pos" "$KEY_LEN")
  best_score=0
  best_shift=0

  for (( shift=0; shift<26; shift++ )); do
    decrypted=$(caesar_decrypt "$subseq" "$shift")
    score=$(score_english "$decrypted")
    if (( score > best_score )); then
      best_score=$score
      best_shift=$shift
    fi
  done

  key_letter=$(printf "$(printf '%03o' $((best_shift + 65)))")
  echo "Best shift: $best_shift → key letter: $key_letter (score: $best_score)"
  KEY+="$key_letter"
done

echo "Recovered key: $KEY"
```

**Recovered key:** `FREKEY`

### Decryption Script (Bash)

```bash
#!/usr/bin/env bash

CIPHER="HCIKV RJOX"
KEY="FREKEY"

decrypt() {
  local cipher="$1"
  local key="$2"
  local klen=${#key}
  local result=""
  local ki=0

  for (( i=0; i<${#cipher}; i++ )); do
    c="${cipher:$i:1}"
    if [[ "$c" =~ [A-Z] ]]; then
      c_val=$(( $(printf '%d' "'$c") - 65 ))
      k_val=$(( $(printf '%d' "'${key:$((ki % klen)):1}") - 65 ))
      p=$(( (c_val - k_val + 26) % 26 ))
      result+=$(printf "$(printf '%03o' $((p + 65)))")
      (( ki++ ))
    else
      result+="$c"
    fi
  done

  echo "$result"
}

decrypt "$CIPHER" "$KEY"
```

> **Password for krypton5:** `CLEARTEXT`

---

## Level 5

When the key length is unknown, statistical methods like the **Kasiski test** or **Index of Coincidence** are used to determine it, then frequency analysis is applied as in Level 4.

> **Password for krypton6:** *(recovered via statistical key-length attack)*

---

## Level 6 

**Files:** `encrypt6`, `keyfile.dat`, `krypton7`

**Ciphertext in krypton7:** `PNUKLYLWRQKGKBE`

### Concept

A stream cipher encrypts one byte at a time using XOR:

```
C = P XOR K
P = C XOR K
```

### Vulnerability

Encrypting 100 `A`s reveals a **repeating pattern every 30 characters**, meaning:

- The keystream repeats
- The RNG is weak and predictable

Since we know the plaintext (`AAAA...`), we can recover the keystream:

```
K = C XOR P
```

### Keystream Recovery (Python)

```python
cipher = b"PNUKLYLWRQKGKBE"
known_plain = b"AAAAAAAAAAAAAAA"

keystream = bytes([c ^ p for c, p in zip(cipher, known_plain)])
print("Recovered keystream:", keystream)
```

### Vigenère Decryption Template (Python)

```python
import string

alphabet = string.ascii_uppercase
cipher = "CIPHERTEXTHERE"
key = "KEY"

result = ""
for i, c in enumerate(cipher):
    if c in alphabet:
        shift = alphabet.index(key[i % len(key)])
        result += alphabet[(alphabet.index(c) - shift) % 26]
    else:
        result += c

print(result)
```

Decrypting `krypton7` with the recovered keystream yields: `LFSRISNOTRANDOM`

> **Password for krypton7:** `LFSRISNOTRANDOM`

---

## Level 7 

🎉 **Congratulations on beating Krypton!**

---

## What Krypton Demonstrates

| Level | Cipher | Attack |
|-------|--------|--------|
| 0 | Base64 | Not encryption — trivially decoded |
| 1 | Caesar | Brute force (only 25 shifts) |
| 2 | Caesar | Known plaintext |
| 3 | Monoalphabetic substitution | Frequency analysis |
| 4 | Vigenère (known key length) | Per-position frequency analysis |
| 5 | Vigenère (unknown key length) | Kasiski / Index of Coincidence |
| 6 | Stream cipher | Weak RNG / keystream reuse |

Modern cryptography avoids these weaknesses through large key sizes, cryptographically secure PRNGs, and strong mathematical foundations (AES, ChaCha20, etc.).
