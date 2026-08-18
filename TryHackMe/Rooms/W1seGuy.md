# W1seGuy
#THM #crypto #easy
## Problem: 
The server provides the flag encrypted and we need to find the key.
## Goal:
Find the encryption key to.
## Technique:
Analyzed the provided code, then used brute force to get the key.
#### Steps:
###### 1) Analyzing the provided code

```python
res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))  
key = str(res)
```
This block generates the encryption key: a random 5 character string made up of letters (upper/lowercase) and digits. 
```python
def setup(server, key):  
    flag = 'THM{thisisafakeflag}'   
    xored = ""  
  
    for i in range(0,len(flag)):  
        xored += chr(ord(flag[i]) ^ ord(key[i%len(key)]))  
  
    hex_encoded = xored.encode().hex()  
    return hex_encoded
```
This function creates the encrypted string: using the XOR operation and hexadecimal transformation.

**Key insight**: since the key repeats (`key[i % len(key)]`) and the flag always starts with `THM{`, we can XOR the first few bytes of the encrypted output against `THM{` to recover most of the key.

###### 2) Connecting to the server and get the encrypted flag
**Code:** `nc 10.128.169.84 1337`

**Response:** `This XOR encoded text has flag 1: 392a3742<--------------------------cencored----------------------------->
What is the encryption key?`
###### 3) Brute forcing the string to get the key using a python script

```python
import random  
import string  
  
ciphertext_hex = "392a3742"  
  
ct_bytes = bytes.fromhex(ciphertext_hex)  
ct_text = ct_bytes.decode('utf-8')  
  
target = "THM{"  
  
alphabet = string.ascii_letters + string.digits  
key_length = 4  
attempts = 0  
  
while True:  
    key = ''.join(random.choices(alphabet, k=key_length))  
    attempts += 1  
    decoded = ''.join(chr(ord(ct_text[i]) ^ ord(key[i % key_length])) for i in range(len(target)))  
  
    if decoded == target:  
        print(f"\nDecoded: {decoded}\n")  
        print(f"Found key: {key}  (after {attempts} attempts)")  
        break  
  
    if attempts % 100000 == 0:  
        print(f"Still trying... {attempts} attempts so far")
```
This script brute-forces a random 4-character key each attempt, decrypts the first 4 bytes of ciphertext (`THM{` encrypted), and checks if the result matches the known `target = "THM{"`. Once found, this gives 4 of the 5 key characters, the 5th still needs to be recovered separately.

> [!NOTE]
> why this works ?
> This works because XOR is reversible — `a ^ b = c` also means `a = b ^ c`.

**Script output:**
```
Decoded: THM{

Found key: mbz9  (after 2572584 attempts)
```
###### 4) Using CyberChef to get the full key
Since the key is 5 characters and `THM{` is 4 characters, the first 4 characters found in step 3 are correct. Plugged the encrypted hex into CyberChef's "From Hex" → "XOR" recipe, then manually tried different 5th characters in the key until the output decoded into a readable flag.

**Key:** `mbz9q`

<img width="1600" height="865" alt="image" src="https://github.com/user-attachments/assets/41d5a110-204e-4a17-bbe9-5870fd0b9a32" />

###### 5) Submitting the key
```bash
This XOR encoded text has flag 1: 392a3742<--------------------------cencored----------------------------->
What is the encryption key? mbz9q                   
Congrats! That is the correct key! Here is flag 2: THM{<FLAG2>}
```
## Root cause:
The XOR key was short (5 characters) and repeated across the whole flag, and the flag format was predictable (`THM{...}`).
## Flag 1:
`THM{<FLAG1>}`
## Flag 2:
`THM{<FLAG2>}`
## Key takeaway:
Weak/repeating-key encryption can be broken when part of the plaintext is known in advance (known-plaintext attack), a fixed, predictable format like `THM{...}` is enough to leak most of the key.
