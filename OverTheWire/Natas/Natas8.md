# Natas8
#natas #crypto
## Vulnerability:
The application stores the secret using reversible reversible encoding instead of secure cryptographic algorithms.
## Technique:
Used python script to decode the secret.
#### Steps:
##### 1) Viewing source code
The source code revealed the algorithm used to store the secret:
```php
<?php

$encodedSecret = "<REDACTED>";

function encodeSecret($secret) {
    return bin2hex(strrev(base64_encode($secret)));
}

if (array_key_exists("submit", $_POST)) {
    if (encodeSecret($_POST['secret']) == $encodedSecret) {
        print "Access granted. The password for natas9 is <censored>";
    } else {
        print "Wrong secret";
    }
}

?>
```
##### 2) Reversing the encoding algorithm
**Script:**
```python
import base64  
secret = "<REDACTED>"  
def decode(string):  
    txt = bytes.fromhex(string).decode()  
    new = txt[::-1]  
    return base64.b64decode(new).decode()  
  
print(decode(secret))
```

##### 3) Submitting the secret

<img width="600" height="222" alt="image" src="https://github.com/user-attachments/assets/be5ed04f-fc26-4c44-ae9b-568bb56d02d9" />


## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas7](./Natas7.md)
## Next Level:
[Natas9](./Natas9.md)
